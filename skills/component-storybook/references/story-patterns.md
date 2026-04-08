# Story Patterns for Common Components

## Button Stories

```tsx
// components/ui/button.stories.tsx
import type { Meta, StoryObj } from "@storybook/react";
import { Button } from "./button";

const meta: Meta<typeof Button> = {
  title: "UI/Button",
  component: Button,
  tags: ["autodocs"],
  argTypes: {
    variant: {
      control: "select",
      options: ["default", "destructive", "outline", "secondary", "ghost", "link"],
    },
    size: { control: "select", options: ["default", "sm", "lg", "icon"] },
    disabled: { control: "boolean" },
  },
};

export default meta;
type Story = StoryObj<typeof Button>;

export const Default: Story = {
  args: { children: "Button", variant: "default" },
};

export const Destructive: Story = {
  args: { children: "Delete", variant: "destructive" },
};

export const Outline: Story = {
  args: { children: "Outline", variant: "outline" },
};

export const WithIcon: Story = {
  render: (args) => (
    <Button {...args}>
      <svg className="mr-2 h-4 w-4" fill="none" viewBox="0 0 24 24" stroke="currentColor">
        <path strokeLinecap="round" strokeLinejoin="round" strokeWidth={2} d="M12 4v16m8-8H4" />
      </svg>
      Add Item
    </Button>
  ),
};

export const Loading: Story = {
  args: { children: "Saving...", disabled: true },
};

export const AllVariants: Story = {
  render: () => (
    <div className="flex flex-wrap gap-4">
      <Button variant="default">Default</Button>
      <Button variant="secondary">Secondary</Button>
      <Button variant="destructive">Destructive</Button>
      <Button variant="outline">Outline</Button>
      <Button variant="ghost">Ghost</Button>
      <Button variant="link">Link</Button>
    </div>
  ),
};
```

## Input Stories

```tsx
// components/ui/input.stories.tsx
import type { Meta, StoryObj } from "@storybook/react";
import { Input } from "./input";
import { Label } from "./label";

const meta: Meta<typeof Input> = {
  title: "UI/Input",
  component: Input,
  tags: ["autodocs"],
  decorators: [(Story) => <div className="w-80"><Story /></div>],
};

export default meta;
type Story = StoryObj<typeof Input>;

export const Default: Story = {
  args: { placeholder: "Enter your email" },
};

export const WithLabel: Story = {
  render: () => (
    <div className="space-y-2">
      <Label htmlFor="email">Email</Label>
      <Input id="email" type="email" placeholder="you@example.com" />
    </div>
  ),
};

export const WithError: Story = {
  render: () => (
    <div className="space-y-2">
      <Label htmlFor="email">Email</Label>
      <Input id="email" type="email" className="border-red-500" aria-invalid />
      <p className="text-sm text-red-500">Please enter a valid email address.</p>
    </div>
  ),
};

export const Disabled: Story = {
  args: { disabled: true, value: "read-only@example.com" },
};
```

## Card Stories

```tsx
// components/ui/card.stories.tsx
import type { Meta, StoryObj } from "@storybook/react";
import { Card, CardHeader, CardTitle, CardDescription, CardContent, CardFooter } from "./card";
import { Button } from "./button";

const meta: Meta<typeof Card> = {
  title: "UI/Card",
  component: Card,
  tags: ["autodocs"],
  decorators: [(Story) => <div className="w-96"><Story /></div>],
};

export default meta;
type Story = StoryObj<typeof Card>;

export const Default: Story = {
  render: () => (
    <Card>
      <CardHeader>
        <CardTitle>Visa Assessment</CardTitle>
        <CardDescription>Check your eligibility for skilled migration.</CardDescription>
      </CardHeader>
      <CardContent>
        <p className="text-sm text-gray-600">
          Answer a few questions to get an instant eligibility assessment.
        </p>
      </CardContent>
      <CardFooter>
        <Button className="w-full">Start Assessment</Button>
      </CardFooter>
    </Card>
  ),
};

export const WithStats: Story = {
  render: () => (
    <Card>
      <CardHeader>
        <CardTitle>Points Score</CardTitle>
        <CardDescription>Your current assessment result</CardDescription>
      </CardHeader>
      <CardContent>
        <div className="text-4xl font-bold text-blue-600">80</div>
        <p className="text-sm text-gray-500">out of 130 points</p>
      </CardContent>
    </Card>
  ),
};
```

## Modal Stories

```tsx
// components/ui/modal.stories.tsx
import type { Meta, StoryObj } from "@storybook/react";
import { useState } from "react";
import { Button } from "./button";

const meta: Meta = {
  title: "UI/Modal",
  tags: ["autodocs"],
};

export default meta;

export const ConfirmDialog: StoryObj = {
  render: () => {
    const [isOpen, setIsOpen] = useState(false);

    return (
      <>
        <Button onClick={() => setIsOpen(true)}>Open Modal</Button>
        {isOpen && (
          <div className="fixed inset-0 z-50 flex items-center justify-center bg-black/50">
            <div className="w-full max-w-md rounded-lg bg-white p-6 shadow-xl">
              <h2 className="text-lg font-semibold">Delete Account?</h2>
              <p className="mt-2 text-sm text-gray-600">This action cannot be undone.</p>
              <div className="mt-4 flex justify-end gap-3">
                <Button variant="outline" onClick={() => setIsOpen(false)}>Cancel</Button>
                <Button variant="destructive" onClick={() => setIsOpen(false)}>Delete</Button>
              </div>
            </div>
          </div>
        )}
      </>
    );
  },
};
```

## Table Stories

```tsx
// components/ui/data-table.stories.tsx
import type { Meta, StoryObj } from "@storybook/react";

const meta: Meta = {
  title: "Data/Table",
  tags: ["autodocs"],
  decorators: [(Story) => <div className="w-full max-w-3xl"><Story /></div>],
};

export default meta;

const SAMPLE_DATA = [
  { id: "1", name: "Software Engineer", code: "261313", points: 80, status: "eligible" },
  { id: "2", name: "Registered Nurse", code: "254499", points: 65, status: "eligible" },
  { id: "3", name: "Chef", code: "351311", points: 55, status: "not_eligible" },
];

export const Default: StoryObj = {
  render: () => (
    <table className="w-full text-sm">
      <thead>
        <tr className="border-b text-left text-gray-500">
          <th className="pb-2">Occupation</th>
          <th className="pb-2">Code</th>
          <th className="pb-2">Points</th>
          <th className="pb-2">Status</th>
        </tr>
      </thead>
      <tbody>
        {SAMPLE_DATA.map((row) => (
          <tr key={row.id} className="border-b">
            <td className="py-3 font-medium">{row.name}</td>
            <td className="py-3">{row.code}</td>
            <td className="py-3">{row.points}</td>
            <td className="py-3">
              <span className={`rounded px-2 py-1 text-xs font-medium ${
                row.status === "eligible" ? "bg-green-100 text-green-800" : "bg-red-100 text-red-800"
              }`}>
                {row.status === "eligible" ? "Eligible" : "Not Eligible"}
              </span>
            </td>
          </tr>
        ))}
      </tbody>
    </table>
  ),
};

export const Empty: StoryObj = {
  render: () => (
    <div className="py-12 text-center text-gray-500">
      No assessments yet. Start your first assessment to see results here.
    </div>
  ),
};

export const Loading: StoryObj = {
  render: () => (
    <div className="space-y-3">
      {[1, 2, 3].map((i) => (
        <div key={i} className="h-12 animate-pulse rounded bg-gray-100" />
      ))}
    </div>
  ),
};
```
