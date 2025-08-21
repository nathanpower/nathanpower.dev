---
title: 'Working with react-query'
published: 2025-07-10
draft: false
description: 'Lessons learned while working with the react-query library from Tanstack'
tags: ['react']
---
```tsx
import { useAxios } from '@myorg/authenticated-axios-wrapper';
import { useQuery } from '@tanstack/react-query';

const useFlorpsQuery = ({ queryParams }: { queryParams: FlorpQueryParams }) => {
  const client = useAxios();
  const { openErrorModal, closeErrorModal } = useErrorModal();
  return useQuery({
    queryKey: ['get-florps', queryParams],
    queryFn: () => client.get<FlorpsListSchema>(FLORPS_API_URL, { params: queryParams })
  });
};

const FlorpsList = () => {
  const { data: florps, isLoading } = useFlorpsQuery({ queryParams: { limit: 10, offset: 0 } });

  if (isLoading || !florps) {
    return <div>Loading...</div>;
  }

  return (
    <ul>
      {florps.map(({ name }) => (
        <li key={name}>{name}</li>
      ))}
    </ul>
  );
};
```

I've been a fan of the `tanstack/react-query` library for a few years, during which time I've occasionally noticed that people new to the library (or new to frontend development in general) question the value it brings. I thought it might be useful to outline some of my thoughts on that subject, as well as some potentially useful learnings acquired since I began using it.

## Why do we need this thing anyway?

Much like React itself, the value TanStack Query brings is most keenly felt if you have ever had to hand-roll solutions in the same problem space. Most people who had to maintain large/complex Query event-driven UIs, or indeed codebases using the 2-way data-binding approaches used by frameworks like Knockout or Angular in the early 2010s, tended to appreciate the one-way data-flow and declarative nature of React when it was released. I had a similar experience reading the first paragraph of the docs for `tanstack/query` (then `react-query` ) because of an experience I had had some years before implementing a data-fetching layer for an application.

In 2018 or so, I was tasked with implementing an infinite-loading style UI for a list-view in a progressive web app, targeting mobile browsers. Initially I had a pretty standard pagination approach, over a virtualised list. The first page would load initially, and further pages were loaded as they scrolled into view. The page size was not fixed, but rather calculated based on the scroll distance. Items were cached (into a `redux` store) once loaded so that you would see previously loaded items when you scrolled back up, but the cache also invalidated after some time, triggering a refresh of the current page.

This was manageable enough, complexity-wise, until we introduced the ability to sort by different attributes of the list items. Because the cache was items-per-page, and pages were associated with scroll windows, now we needed a separate cache for each sort value.

Along with this, we also found that de-duplication of requests and limiting requests in flight in general was necessary for the more enthusiastic scrollers out there - it became more of a maintenance burden than we would have liked. Fast-forward a few years, I am on a different project, pondering another data-layer, and I come across this library and the following claim:

> React Query is often described as the missing data-fetching library for React, but in more technical terms, it makes **fetching, caching, synchronizing and updating server state** in your React applications a breeze.

This paragraph was enough for me to hack together a quick proof-of-concept, and I was pretty convinced. It has since became the de-facto way to handle client-side data requirements in the organisation.

## So what does it do?

TanStack Query is responsible for keeping the client-side representation of server state up to date, and triggering re-renders of the React tree when it updates. This server data is kept in memory in a key-value form referred to as the `query-cache`. This means that the developer can offload the following concerns to the library (from the [official docs](https://tanstack.com/query/latest/docs/framework/react/overview)):

- Caching... (possibly the hardest thing to do in programming)
- De-duping multiple requests for the same data into a single request
- Updating "out of date" data in the background
- Knowing when data is "out of date"
- Reflecting updates to data as quickly as possible
- Performance optimisations like pagination and lazy loading data
- Managing memory and garbage collection of server state
- Memoizing query results with structural sharing

## What does it not do?

- Actual network requests (bring your own HTTP client)
- Synchronous state updates

## Recommended usage patterns

What follows is some opinionated advice based on using the library for a number of use-cases over 2 years or so.

### Cache the data you use, not the data you fetch

If you have expensive business logic that must transform the server response before rendering to the UI, it's best that this happens between the fetch and the cache layer, otherwise it will run every time your data is returned from the cache aka on every render that reads it. This can significantly improve performance and responsiveness of your interfaces. It's worth noting here that the `select` function runs _post-cache_ - so put this expensive logic inside your `queryFn`. On a related note: nothing says that you only have to make one request inside a `queryFn`. You are free to make many requests, and combine them as you see fit, before resolving the promise returned.

### Wrap useQuery / useMutation in a custom function per resource

Lets say you have an endpoint that returns `florps`. A good level of abstraction to aim for is `useFlorpsQuery`. We have found that trying to make functions more generic than this is not worth the hassle involved with keeping things type-safe, and it inevitably ends up having too many options exposed, making it harder to change and harder to maintain.

### Use the official development tooling

Two very useful tools are available, the `ReactQueryDevTools` component, which overlays some floating UI to give visual insights into your query keys and the state of your cache, and an ESLint plugin which will keep devs to best practices and avoid footguns (like unstable object references or excessive re-renders).

- [React Query DevTools](https://tanstack.com/query/v5/docs/framework/react/devtools)
- [ESLint Plugin Query](https://tanstack.com/query/latest/docs/eslint/eslint-plugin-query)

### Rely on inference, rather than passing generic type arguments

Passing generic types for `useQuery` or `useMutation` is very easy to get wrong, and is usually not required in order for the return type to be correctly typed, which is _really_ what you want after all. The trick is to strongly type the return of `queryFn` or `mutationFn`, and all types should flow correctly from there. See the [excellent article](https://tkdodo.eu/blog/react-query-and-type-script#type-inference) from Dominik Dorfmeister on this subject.

### Become familiar with the opinionated defaults

It's really worth taking the time to learn how the [default configuration](https://tkdodo.eu/blog/practical-react-query#the-defaults-explained) behaves, and what your options are with regards to tuning for you own use-case. In particular, the default options may be cause issues if your endpoints are performing expensive operations.

For example:

- Query instances via **`useQuery`** or **`useInfiniteQuery`** by default **consider cached data as stale**.
- Stale queries are refetched automatically in the background when:
  - New instances of the query mount
  - The window is refocused
  - The network is reconnected
  - The query is optionally configured with a refetch interval

_Remember that you can set application-wide defaults via the [defaultOptions on QueryClient](https://tanstack.com/query/v4/docs/reference/QueryClient#queryclient)_

### Share types with backend

If you are not validating the response with something like `zod`, an approach we have found to work well is to generate TypeScript types from the API endpoints, and use those to strongly type the return value of `queryFn` or `mutationFn`. We have a Python / Django backend, so we use a combination of [pydantic](https://docs.pydantic.dev/latest/) to define the response types (and validate them at runtime) and [openapi](https://www.npmjs.com/package/openapi) to generate the TypeScript. This gets a bit trickier if the API endpoints do not share a monorepo with the frontend code (as ours does), but it is still doable with some extra steps to ensure backwards compatability. Most server frameworks will have a way to generate an OpenAPI representation of your endpoints.

### Use Mock Service Worker to mock component data dependencies in Storybook

Sometimes it's nice for a reusable component to own it's own data dependencies. That way, it can be used in various places in the app without the parent components needing to know about the data it needs, or where to fetch it.
This can make authoring Storybook representations of the component a bit awkward, as now there component needs to make a HTTP request for it's data, rather than having them passed via props.
The [msw](https://mswjs.io/) library solves this neatly by mocking such requests at the network layer (rather than having to monkey patch `axios` or `fetch`), and, for good measure, the mock handlers exposed can also be re-used by integration-style unit tests.

### Use QueryClient API to ensure better server sync and reduce requests

- Use [invalidateQueries](https://tanstack.com/query/latest/docs/framework/react/guides/query-invalidation) when you know better than `react-query` that things need to be refetched, or call [refetchQueries](https://tanstack.com/query/latest/docs/reference/QueryClient#queryclientrefetchqueries) directly.

- To keep requests to a minimum, consider infinite `staleTime` for queries you know will not change for the duration of the session.

- Make params passed to `useMutations`/ `useQuery` form part of the query key (e.g. `searchTerm`)

## Possible to use as a state manager if you need some global state that does not come from the server

- Set the query-cache value directly and it will be accessible to anything with access to the `queryClient`
- If you are doing this a lot - probably worth thinking about a more purpose-built tool.

As usual Dominik Dorfmeister has [a good article on this](https://tkdodo.eu/blog/react-query-as-a-state-manager)

### Further reading

[Official docs](https://tanstack.com/query/latest/docs/framework/react/overview)

[Dominik Dorfmeister’s blog](https://tkdodo.eu/blog/practical-react-query)
