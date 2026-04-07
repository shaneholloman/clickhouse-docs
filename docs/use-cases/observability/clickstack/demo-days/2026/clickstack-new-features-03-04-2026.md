---
slug: /use-cases/observability/clickstack/demo-days/2026/04/03-04-2026
title: 'Demo days - 03/04/2026'
sidebar_label: '03/04/2026'
pagination_prev: null
pagination_next: null
description: 'ClickStack demo days for 03/04/2026'
doc_type: 'guide'
keywords: ['ClickStack', 'Demo days']
---

## Dashboard and Saved Search Listing Pages {#dashboard-and-saved-search-listing-pages}

*Demo by [@pulpdrew](https://github.com/pulpdrew)*

<iframe width="768" height="432" src="https://www.youtube.com/embed/12UNERiTQNE" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" referrerpolicy="strict-origin-when-cross-origin" allowfullscreen></iframe>

The sidebar approach to dashboards and saved searches works fine for small teams, but once you have a lot of them it quickly becomes unmanageable. This update introduces dedicated listing pages for both dashboards and saved searches, with card and list views, tag-based filtering, and name search so you can find what you're looking for fast.

A new favoriting feature lets you pin the dashboards and saved searches you actually use day-to-day, surfacing them at the top of the listing page and in the sidebar for quick access. Saved searches also now have an editable title and breadcrumb navigation for consistency with dashboards, plus a "save as new search" option so you can branch off a search without overwriting the original.

On the dashboard side there's also a new template gallery with four ready-to-import dashboards covering Node.js runtime metrics. Tags can be assigned during import and the whole process takes just a couple of clicks.

**Related PRs:** [#1971](https://github.com/hyperdxio/hyperdx/pull/1971) Add dashboard listing page, [#2021](https://github.com/hyperdxio/hyperdx/pull/2021) Add favorites for dashboards and saved searches, [#2033](https://github.com/hyperdxio/hyperdx/pull/2033) Group dashboards and searches by tag, [#2010](https://github.com/hyperdxio/hyperdx/pull/2010) Add dashboard template gallery, [#2031](https://github.com/hyperdxio/hyperdx/pull/2031) Show created/updated metadata for saved searches and dashboards

## Filters for Filters {#filters-for-filters}

*Demo by [@pulpdrew](https://github.com/pulpdrew)*

<iframe width="768" height="432" src="https://www.youtube.com/embed/kMI3pln2q2s" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" referrerpolicy="strict-origin-when-cross-origin" allowfullscreen></iframe>

Dashboard filter dropdowns now support conditions, so you can scope what values appear in a filter based on other criteria. In the demo, a "service name" dropdown is configured to only show services running Node.js, keeping the list focused and relevant rather than showing every service across the whole stack.

Filter dropdowns are also now multi-select, so you can pick multiple values at once. This is particularly useful on dashboards where you want to compare behaviour across several services or environments without switching views.

**Related PRs:** [#1969](https://github.com/hyperdxio/hyperdx/pull/1969) Add conditions to dashboard filters and support filter multi-select

## RBAC for Predefined Dashboards {#rbac-for-predefined-dashboards}

<iframe width="768" height="432" src="https://www.youtube.com/embed/yZbnsZneSPE" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" referrerpolicy="strict-origin-when-cross-origin" allowfullscreen></iframe>

Role-based access control is now enforced on the predefined/preset dashboards, closing a gap where those dashboards were previously visible regardless of a user's permissions. Fine-grained controls are supported, so a role with read-only access to specific services will only see the relevant preset dashboard and cannot modify its filters.

In the demo, a custom role is configured with a filter restricting access to the "services" data. When logging in as a user with that role, only the services dashboard appears, and the filters on that dashboard are locked to read-only, matching the permissions set for the role.

## Optimizations for Searching {#optimizations-for-searching}

*Demo by [@knudtty](https://github.com/knudtty)*

<iframe width="768" height="432" src="https://www.youtube.com/embed/0hYqJHOWixk" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" referrerpolicy="strict-origin-when-cross-origin" allowfullscreen></iframe>

ClickHouse has a feature called Read in Order, which lets queries with an ORDER BY matching the table's primary key read data sequentially and stop early once a LIMIT is satisfied. Search queries were already using this, but benchmarking revealed they were still over-fetching data on large tables with many parts to iterate over.

The fix is a small but effective one: for search queries, a one-minute pre-window is prepended to the normal windowed query sequence. Most searches have data in the most recent minute anyway, so this short initial scan returns results almost immediately for the common case. If nothing is found in that window, the query falls through to the standard approach as normal.

The result is noticeably faster searches across large datasets, with no change to how searches behave from a user perspective.

**Related PRs:** [#2019](https://github.com/hyperdxio/hyperdx/pull/2019) Use 1 minute window for searches

## Copy Row and Configurable Filter Sizes {#copy-row-and-configurable-filter-sizes}

*Demo by [@knudtty](https://github.com/knudtty)*

<iframe width="768" height="432" src="https://www.youtube.com/embed/MttfyZUbBAE" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" referrerpolicy="strict-origin-when-cross-origin" allowfullscreen></iframe>

A copy button has been added to the results table and the full sidebar view, letting you grab an entire log row as JSON in one click. This is handy when you want to paste a log entry into an AI tool or share it with a teammate without manually selecting fields.

Two other quality-of-life changes are included here. First, the ClickStack beta feature flag has been removed: when you bring up a fresh local instance, ClickStack now appears in the sidebar automatically without any manual flag toggling. Second, a new team setting lets you configure how many filter key values are fetched in filter dropdowns. Previously this was capped at 16, which was too low for larger datasets. With the setting bumped up, filter dropdowns now surface far more attribute values, making it easier to find what you're filtering on.

**Related PRs:** [#2035](https://github.com/hyperdxio/hyperdx/pull/2035) Add copy row as JSON button, [#2020](https://github.com/hyperdxio/hyperdx/pull/2020) New team setting for number of filters to fetch

## Tabs and Groups in Dashboards {#tabs-and-groups-in-dashboards}

*Demo by [@alex-fedotyev](https://github.com/alex-fedotyev)*

<iframe width="768" height="432" src="https://www.youtube.com/embed/N8kRln-Iecc" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" referrerpolicy="strict-origin-when-cross-origin" allowfullscreen></iframe>

A new "group" container type is being introduced for dashboards, giving you a way to organise panels into sections that can be either collapsible or tabbed. Rather than having two separate container types (groups and sections), this consolidates everything into a single concept where you pick the behaviour you want after the fact.

Groups can be configured as collapsible panels, tabbed views, or both, and you can toggle the border on or off depending on how you want the layout to look. Panels can be rearranged within groups, and the whole thing is designed to make preset dashboards more flexible and adaptable rather than locked to a fixed layout.

This is still in progress and being iterated on based on feedback, with the goal of making it easier to build dense, well-organised dashboards without having to think too hard about the container hierarchy.

**Related PRs:** [#1972](https://github.com/hyperdxio/hyperdx/pull/1972) Dashboard groups with tabs, collapsible/bordered options, and panel organisation

## ClickStack CLI {#clickstack-cli}

*Demo by [@wrn14897](https://github.com/wrn14897)*

<iframe width="768" height="432" src="https://www.youtube.com/embed/nfYX47fTg1c" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" referrerpolicy="strict-origin-when-cross-origin" allowfullscreen></iframe>

The new `@hyperdx/cli` package brings a full terminal UI for searching and tailing logs directly from your terminal. You authenticate once with your API endpoint and email, and from there you get the same search experience as the web app, including the familiar search syntax, log entry inspection with column values, and even trace waterfall views, all rendered in the terminal.

The CLI also opens up some interesting possibilities for AI agents. Because it exposes the ClickHouse schema and query interface, an agent can be given the schema output and taught to generate queries directly against it. In the demo, an agent is shown using the CLI to investigate issues from the past two hours, pulling traces and figuring out what went wrong without any manual querying.

Taking that further, because the CLI uses a web session, it can also hand off to Playwright to visit live dashboards and inspect charts visually. The demo shows an agent using this to look up ClickHouse metrics in HyperDX and reason about what it's seeing. The CLI is being packaged for npm and should be available soon.

**Related PRs:** [#2043](https://github.com/hyperdxio/hyperdx/pull/2043) Add @hyperdx/cli package with terminal TUI
