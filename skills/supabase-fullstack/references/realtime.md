# Supabase Realtime

Supabase Realtime lets you listen for changes in your database, broadcast messages between clients, and track who is online. All of this happens over WebSockets.

## Setup: Enable Realtime on a Table

In the Supabase dashboard: go to Database > Tables > your table > Enable Realtime.

Or via SQL:

```sql
ALTER PUBLICATION supabase_realtime ADD TABLE messages;
ALTER PUBLICATION supabase_realtime ADD TABLE notifications;
```

## Postgres Changes (Database Listeners)

Listen for INSERT, UPDATE, DELETE on a table in real-time.

### Listen for New Rows

```tsx
// app/components/live-messages.tsx
"use client";

import { useEffect, useState } from "react";
import { createClient } from "@/lib/supabase/client";

interface Message {
  id: string;
  body: string;
  user_id: string;
  created_at: string;
}

export function LiveMessages({ channelId }: { channelId: string }) {
  const [messages, setMessages] = useState<Message[]>([]);

  useEffect(() => {
    const supabase = createClient();

    // Load existing messages
    supabase
      .from("messages")
      .select("*")
      .eq("channel_id", channelId)
      .order("created_at", { ascending: true })
      .limit(50)
      .then(({ data, error }) => {
        if (!error && data) setMessages(data);
      });

    // Listen for new messages
    const channel = supabase
      .channel(`messages:${channelId}`)
      .on(
        "postgres_changes",
        {
          event: "INSERT",
          schema: "public",
          table: "messages",
          filter: `channel_id=eq.${channelId}`,
        },
        (payload) => {
          setMessages((prev) => [...prev, payload.new as Message]);
        }
      )
      .subscribe();

    return () => {
      supabase.removeChannel(channel);
    };
  }, [channelId]);

  return (
    <div className="space-y-2">
      {messages.map((msg) => (
        <div key={msg.id} className="rounded bg-gray-50 p-3">
          <p>{msg.body}</p>
          <span className="text-xs text-gray-400">
            {new Date(msg.created_at).toLocaleTimeString()}
          </span>
        </div>
      ))}
    </div>
  );
}
```

### Listen for All Change Types

```tsx
// app/components/live-todos.tsx
"use client";

import { useEffect, useState } from "react";
import { createClient } from "@/lib/supabase/client";

interface Todo {
  id: string;
  title: string;
  completed: boolean;
}

export function LiveTodos() {
  const [todos, setTodos] = useState<Todo[]>([]);

  useEffect(() => {
    const supabase = createClient();

    // Initial load
    supabase
      .from("todos")
      .select("*")
      .order("created_at")
      .then(({ data }) => {
        if (data) setTodos(data);
      });

    const channel = supabase
      .channel("todos-all")
      .on(
        "postgres_changes",
        { event: "INSERT", schema: "public", table: "todos" },
        (payload) => {
          setTodos((prev) => [...prev, payload.new as Todo]);
        }
      )
      .on(
        "postgres_changes",
        { event: "UPDATE", schema: "public", table: "todos" },
        (payload) => {
          setTodos((prev) =>
            prev.map((t) => (t.id === payload.new.id ? (payload.new as Todo) : t))
          );
        }
      )
      .on(
        "postgres_changes",
        { event: "DELETE", schema: "public", table: "todos" },
        (payload) => {
          setTodos((prev) => prev.filter((t) => t.id !== payload.old.id));
        }
      )
      .subscribe();

    return () => {
      supabase.removeChannel(channel);
    };
  }, []);

  return (
    <ul className="space-y-2">
      {todos.map((todo) => (
        <li key={todo.id} className={`p-2 ${todo.completed ? "line-through text-gray-400" : ""}`}>
          {todo.title}
        </li>
      ))}
    </ul>
  );
}
```

## Broadcast (Client-to-Client Messages)

Broadcast lets clients send messages to each other without storing anything in the database. Great for cursors, typing indicators, and temporary state.

### Typing Indicator

```tsx
// app/components/typing-indicator.tsx
"use client";

import { useEffect, useState, useCallback } from "react";
import { createClient } from "@/lib/supabase/client";

export function TypingIndicator({
  channelId,
  userId,
  userName,
}: {
  channelId: string;
  userId: string;
  userName: string;
}) {
  const [typingUsers, setTypingUsers] = useState<string[]>([]);

  useEffect(() => {
    const supabase = createClient();

    const channel = supabase
      .channel(`typing:${channelId}`)
      .on("broadcast", { event: "typing" }, (payload) => {
        const { user_name, is_typing } = payload.payload;

        setTypingUsers((prev) => {
          if (is_typing && !prev.includes(user_name)) {
            return [...prev, user_name];
          }
          if (!is_typing) {
            return prev.filter((u) => u !== user_name);
          }
          return prev;
        });
      })
      .subscribe();

    return () => {
      supabase.removeChannel(channel);
    };
  }, [channelId]);

  const sendTyping = useCallback(
    (isTyping: boolean) => {
      const supabase = createClient();
      const channel = supabase.channel(`typing:${channelId}`);

      channel.send({
        type: "broadcast",
        event: "typing",
        payload: { user_name: userName, is_typing: isTyping },
      });
    },
    [channelId, userName]
  );

  // Show typing indicator
  if (typingUsers.length === 0) return null;

  const text =
    typingUsers.length === 1
      ? `${typingUsers[0]} is typing...`
      : `${typingUsers.join(", ")} are typing...`;

  return <p className="text-sm italic text-gray-400">{text}</p>;
}
```

### Live Cursors

```tsx
// app/components/live-cursors.tsx
"use client";

import { useEffect, useState } from "react";
import { createClient } from "@/lib/supabase/client";

interface CursorPosition {
  userId: string;
  userName: string;
  x: number;
  y: number;
}

export function LiveCursors({
  roomId,
  userId,
  userName,
}: {
  roomId: string;
  userId: string;
  userName: string;
}) {
  const [cursors, setCursors] = useState<Map<string, CursorPosition>>(new Map());

  useEffect(() => {
    const supabase = createClient();

    const channel = supabase
      .channel(`cursors:${roomId}`)
      .on("broadcast", { event: "cursor" }, (payload) => {
        const cursor = payload.payload as CursorPosition;
        if (cursor.userId !== userId) {
          setCursors((prev) => new Map(prev).set(cursor.userId, cursor));
        }
      })
      .subscribe();

    function handleMouseMove(e: MouseEvent) {
      channel.send({
        type: "broadcast",
        event: "cursor",
        payload: { userId, userName, x: e.clientX, y: e.clientY },
      });
    }

    window.addEventListener("mousemove", handleMouseMove);

    return () => {
      window.removeEventListener("mousemove", handleMouseMove);
      supabase.removeChannel(channel);
    };
  }, [roomId, userId, userName]);

  return (
    <>
      {Array.from(cursors.values()).map((cursor) => (
        <div
          key={cursor.userId}
          className="pointer-events-none fixed z-50"
          style={{ left: cursor.x, top: cursor.y }}
        >
          <div className="h-4 w-4 rotate-[-30deg] border-2 border-blue-500 bg-blue-200" />
          <span className="ml-4 rounded bg-blue-500 px-2 py-1 text-xs text-white">
            {cursor.userName}
          </span>
        </div>
      ))}
    </>
  );
}
```

## Presence (Who is Online)

Presence tracks which users are currently connected.

```tsx
// app/components/online-users.tsx
"use client";

import { useEffect, useState } from "react";
import { createClient } from "@/lib/supabase/client";

interface OnlineUser {
  userId: string;
  userName: string;
  onlineAt: string;
}

export function OnlineUsers({
  roomId,
  userId,
  userName,
}: {
  roomId: string;
  userId: string;
  userName: string;
}) {
  const [users, setUsers] = useState<OnlineUser[]>([]);

  useEffect(() => {
    const supabase = createClient();

    const channel = supabase
      .channel(`room:${roomId}`, {
        config: { presence: { key: userId } },
      })
      .on("presence", { event: "sync" }, () => {
        const state = channel.presenceState<OnlineUser>();
        const onlineUsers: OnlineUser[] = [];

        Object.values(state).forEach((presences) => {
          presences.forEach((presence) => {
            onlineUsers.push(presence);
          });
        });

        setUsers(onlineUsers);
      })
      .subscribe(async (status) => {
        if (status === "SUBSCRIBED") {
          await channel.track({
            userId,
            userName,
            onlineAt: new Date().toISOString(),
          });
        }
      });

    return () => {
      supabase.removeChannel(channel);
    };
  }, [roomId, userId, userName]);

  return (
    <div className="space-y-1">
      <h3 className="text-sm font-medium text-gray-500">Online ({users.length})</h3>
      {users.map((user) => (
        <div key={user.userId} className="flex items-center gap-2">
          <div className="h-2 w-2 rounded-full bg-green-500" />
          <span className="text-sm">{user.userName}</span>
        </div>
      ))}
    </div>
  );
}
```

## Chat Room Example (Combining Everything)

```tsx
// app/chat/[roomId]/page.tsx
import { createClient } from "@/lib/supabase/server";
import { redirect } from "next/navigation";
import { ChatRoom } from "./chat-room";

export default async function ChatPage({
  params,
}: {
  params: Promise<{ roomId: string }>;
}) {
  const { roomId } = await params;
  const supabase = await createClient();
  const { data: { user } } = await supabase.auth.getUser();

  if (!user) redirect("/login");

  const { data: profile } = await supabase
    .from("profiles")
    .select("full_name")
    .eq("id", user.id)
    .single();

  // Load initial messages on the server
  const { data: messages } = await supabase
    .from("messages")
    .select("id, body, user_id, created_at, profiles(full_name)")
    .eq("channel_id", roomId)
    .order("created_at", { ascending: true })
    .limit(100);

  return (
    <ChatRoom
      roomId={roomId}
      userId={user.id}
      userName={profile?.full_name ?? "Anonymous"}
      initialMessages={messages ?? []}
    />
  );
}
```

```tsx
// app/chat/[roomId]/chat-room.tsx
"use client";

import { useEffect, useState, useRef } from "react";
import { createClient } from "@/lib/supabase/client";

interface Message {
  id: string;
  body: string;
  user_id: string;
  created_at: string;
  profiles: { full_name: string } | null;
}

export function ChatRoom({
  roomId,
  userId,
  userName,
  initialMessages,
}: {
  roomId: string;
  userId: string;
  userName: string;
  initialMessages: Message[];
}) {
  const [messages, setMessages] = useState<Message[]>(initialMessages);
  const [input, setInput] = useState("");
  const bottomRef = useRef<HTMLDivElement>(null);
  const supabase = createClient();

  useEffect(() => {
    const channel = supabase
      .channel(`chat:${roomId}`)
      .on(
        "postgres_changes",
        {
          event: "INSERT",
          schema: "public",
          table: "messages",
          filter: `channel_id=eq.${roomId}`,
        },
        async (payload) => {
          // Fetch the full message with profile
          const { data } = await supabase
            .from("messages")
            .select("id, body, user_id, created_at, profiles(full_name)")
            .eq("id", payload.new.id)
            .single();

          if (data) {
            setMessages((prev) => [...prev, data]);
          }
        }
      )
      .subscribe();

    return () => {
      supabase.removeChannel(channel);
    };
  }, [roomId, supabase]);

  useEffect(() => {
    bottomRef.current?.scrollIntoView({ behavior: "smooth" });
  }, [messages]);

  async function handleSend(e: React.FormEvent) {
    e.preventDefault();
    if (!input.trim()) return;

    const { error } = await supabase.from("messages").insert({
      body: input.trim(),
      channel_id: roomId,
      user_id: userId,
    });

    if (error) {
      alert("Failed to send message");
      return;
    }

    setInput("");
  }

  return (
    <div className="flex h-screen flex-col">
      <div className="flex-1 overflow-y-auto p-4 space-y-3">
        {messages.map((msg) => (
          <div
            key={msg.id}
            className={`max-w-[70%] rounded-lg p-3 ${
              msg.user_id === userId
                ? "ml-auto bg-blue-500 text-white"
                : "bg-gray-100"
            }`}
          >
            <p className="text-xs font-medium opacity-70">
              {msg.profiles?.full_name ?? "Unknown"}
            </p>
            <p>{msg.body}</p>
          </div>
        ))}
        <div ref={bottomRef} />
      </div>

      <form onSubmit={handleSend} className="border-t p-4 flex gap-2">
        <input
          value={input}
          onChange={(e) => setInput(e.target.value)}
          placeholder="Type a message..."
          className="flex-1 rounded border p-3"
        />
        <button type="submit" className="rounded bg-blue-500 px-6 py-3 text-white">
          Send
        </button>
      </form>
    </div>
  );
}
```

## Cleanup

Always remove channels when the component unmounts to prevent memory leaks:

```tsx
useEffect(() => {
  const channel = supabase.channel("my-channel").subscribe();

  return () => {
    supabase.removeChannel(channel); // Clean up
  };
}, []);
```
