---
title: 'Composable component APIs'
published: 2025-07-10
draft: false
description: 'Should your internal component library have a composable API'
tags: ['react']
---
For those of us who have followed the evolution of UI component libraries in the React ecosystem, it's hard not be inspired by the elegant composable APIs exemplified by `react-aria` and `radix-ui`. Is this also the approach we should follow when authoring internal component libraries, for example internal design-system implementations?

```tsx
import React from 'react';
import * as Select from '@radix-ui/react-select';
import classnames from 'classnames';
import { CheckIcon, ChevronDownIcon, ChevronUpIcon } from '@radix-ui/react-icons';
import './styles.css';

const SelectDemo = () => (
  <Select.Root>
    <Select.Trigger className="SelectTrigger" aria-label="Food">
      <Select.Value placeholder="Select a fruit…" />
      <Select.Icon className="SelectIcon">
        <ChevronDownIcon />
      </Select.Icon>
    </Select.Trigger>
    <Select.Portal>
      <Select.Content className="SelectContent">
        <Select.ScrollUpButton className="SelectScrollButton">
          <ChevronUpIcon />
        </Select.ScrollUpButton>
        <Select.Viewport className="SelectViewport">
          <Select.Group>
            <Select.Label className="SelectLabel">Fruits</Select.Label>
            <SelectItem value="apple">Apple</SelectItem>
            <SelectItem value="banana">Banana</SelectItem>
            <SelectItem value="blueberry">Blueberry</SelectItem>
            <SelectItem value="grapes">Grapes</SelectItem>
            <SelectItem value="pineapple">Pineapple</SelectItem>
          </Select.Group>
      </Select.Content>
    </Select.Portal>
  </Select.Root>
);
```

Isn't that lovely? That's example is copied and condensed from the [radix-ui examples](https://www.radix-ui.com/primitives/docs/components/select). Clear hierarchy, flexibility via composition, looks almost like HTML. Right? RIGHT?

Well...

Much as I like it, I am not convinced it is the style I would recommend exposing to teams, particularly if you have to support developers with limited frontend experience.

Let me outline some of the reasons why:

## Rules of composition are opaque

When you look at the Radix Select example above, it's not immediately obvious which components are required, which are optional, or what the valid nesting relationships are. Can `Select.Icon` exist outside of `Select.Trigger`? What happens if you forget `Select.Portal`?

These rules exist in the documentation and TypeScript definitions, but they're not self-evident from the API surface. A developer needs to understand the mental model of how these pieces fit together, which adds cognitive overhead.

Compare this to a more traditional API:

```tsx
<Select
  options={fruits}
  placeholder="Select a fruit…"
  icon={<ChevronDownIcon />}
/>
```

The constraints are clear: you pass props, you get a select. No composition rules to remember.

## Behaviour of composed children is opaque

In composable APIs, child components often receive props or context invisibly from their parents. In our Radix example, `Select.Value` somehow knows what the current selection is, and `Select.Item` components can update that selection - but this magic happens behind the scenes.

This can lead to confusing debugging sessions. When something doesn't work as expected, developers need to understand not just their own code, but the implicit data flow between composed components.

```tsx
// Why doesn't this work? What's missing?
<Select.Root>
  <Select.Trigger>
    <Select.Value /> {/* No placeholder appears */}
  </Select.Trigger>
</Select.Root>
```

The answer might be that you need `defaultValue` on `Select.Root`, or that `Select.Value` needs a `placeholder` prop, but this isn't obvious from the component structure alone.

## Internal APIs sometimes benefit from opinions over flexibility

Public libraries like Radix need to serve diverse use cases across thousands of applications. They optimize for flexibility because they can't predict every scenario their users will encounter.

Internal component libraries have different constraints. You typically know:
- Your design system's specific patterns
- Your team's skill levels and preferences
- The specific use cases you need to support
- Your organization's accessibility and testing requirements

This knowledge lets you make opinionated choices that reduce complexity:

```tsx
// Opinionated: always includes error state, loading state, proper labeling
<FormSelect
  label="Favorite fruit"
  options={fruits}
  value={selectedFruit}
  onChange={setSelectedFruit}
  error={validationError}
  loading={isLoading}
  required
/>
```

Rather than requiring developers to compose these concerns themselves:

```tsx
<Select.Root value={selectedFruit} onValueChange={setSelectedFruit}>
  <Label htmlFor="fruit-select" required>Favorite fruit</Label>
  <Select.Trigger id="fruit-select" disabled={isLoading}>
    {isLoading ? <Spinner /> : <Select.Value />}
    <Select.Icon><ChevronDownIcon /></Select.Icon>
  </Select.Trigger>
  {/* ... rest of composition */}
  {validationError && <ErrorMessage>{validationError}</ErrorMessage>}
</Select.Root>
```

## Finding the right balance

This doesn't mean composable APIs are always wrong for internal libraries. They excel when you need:
- Complex, varied layouts that can't be anticipated
- Components that need to integrate with diverse parent contexts
- Maximum flexibility for power users

But for most internal design systems, I'd recommend starting with simpler, more opinionated APIs and only introducing composition where the flexibility is genuinely needed. Your future self (and your teammates) will thank you for the reduced cognitive load.

The goal isn't to build the most elegant API - it's to build the most effective one for your team and context.
