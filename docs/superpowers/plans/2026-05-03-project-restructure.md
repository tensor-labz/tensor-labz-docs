# Project Restructure — Decoupled Architecture Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Refactor the codebase from a Context-API-everything monolith into a professional, decoupled architecture with a pure service layer, Redux Toolkit for remote state, feature-colocated UI + controller hooks, and a clean shared layer.

**Architecture:** Remote data (hero slides, services, projects, about, contact) moves from React Context into Redux Toolkit slices fed by a pure `services/` layer. UI-only state (theme, device size) stays in Context. Each page feature owns its components + a controller hook that bridges the store to the view — no fetch logic in components.

**Tech Stack:** React 18, TypeScript strict, Redux Toolkit 2.x, React Redux 9.x, Vite 6, Tailwind CSS, Motion/React

---

## Target File Structure

```
src/
├── app/
│   ├── store.ts            ← Redux store (root reducer + middleware)
│   ├── hooks.ts            ← Typed useAppDispatch / useAppSelector
│   └── providers.tsx       ← All providers composed (replaces RootContext)
│
├── services/               ← Pure async fetch functions — no React, no state
│   ├── sheetsClient.ts     ← Base fetch helper + URL config
│   ├── heroService.ts      ← fetchHeroSlides()
│   ├── servicesService.ts  ← fetchServices()
│   ├── projectsService.ts  ← fetchProjects()
│   ├── aboutService.ts     ← fetchAboutData()
│   └── contactService.ts   ← fetchContactData()
│
├── store/                  ← Redux slices (state + thunks + selectors)
│   ├── heroSlice.ts
│   ├── servicesSlice.ts
│   ├── projectsSlice.ts
│   ├── aboutSlice.ts
│   └── contactSlice.ts
│
├── features/               ← Feature modules: UI components + controller hooks
│   ├── hero/
│   │   ├── components/     ← HeroSection, HeroImageSlider, HeroKeyPoint, HeroLoading
│   │   └── hooks/useHeroController.ts
│   ├── home-services/
│   │   ├── components/     ← ServiceSection, ServiceCard
│   │   └── hooks/useHomeServicesController.ts
│   ├── home-projects/
│   │   ├── components/     ← LatestProductSection, LatestProductCard, MobileCarousel, LatestHero, TopProductTitle
│   │   └── hooks/useLatestProjectsController.ts
│   ├── services-page/
│   │   ├── components/     ← ServiceHero, ServiceContainer, ProjectCard, TabBar, ServiceDropDown, Pagination, ServiceEmpty, ServiceLoading
│   │   └── hooks/useServicesPageController.ts
│   ├── project-detail/
│   │   ├── components/     ← ProjectHero, ProjectNotFound, ProjectPageLoading
│   │   └── hooks/useProjectDetailController.ts
│   ├── about/
│   │   ├── components/     ← AboutUsSection, AboutLoading
│   │   └── hooks/useAboutController.ts
│   └── contact/
│       ├── components/     ← ContactInfo, SocialMediaLinks, ContactUsPlaceholder
│       └── hooks/useContactController.ts
│
├── shared/
│   ├── components/
│   │   ├── ui/             ← Card, Section, PageWrapper (renamed from resuable/)
│   │   ├── layout/         ← Header, Footer, NavBar, NavItem, MobileNavigation, Layout
│   │   ├── feedback/       ← ErrorBoundary, OfflineWarning
│   │   └── three/          ← GlobalBackground
│   ├── hooks/
│   │   ├── useScroll.ts    ← (moved from base/hooks/)
│   │   ├── useDevice.ts    ← (extracted from DeviceContext)
│   │   └── useTheme.ts     ← (extracted from ThemeContext)
│   ├── types/
│   │   ├── common.ts       ← ParentType, AnimateVariables, LinkProps, EventProps, ImageProps
│   │   ├── service.ts      ← ServiceCardProps
│   │   ├── project.ts      ← ProjectItem, ProjectProps
│   │   └── socialMedia.ts  ← SocialMediaItem
│   └── utils/
│       ├── throttle.ts     ← (moved from base/util/)
│       └── googleDrive.ts  ← extractGoogleDriveFileId (moved from base/hooks/)
│
├── pages/                  ← Thin route-level wrappers (assemble features)
│   ├── HomePage.tsx
│   ├── AboutPage.tsx
│   ├── ContactPage.tsx
│   ├── ServicesPage.tsx
│   └── ProjectPage.tsx
│
├── data/                   ← Static config only
│   ├── app.ts              ← (renamed from app_data.tsx)
│   ├── nav.ts              ← (renamed from nav_data.tsx)
│   └── static.ts           ← (renamed from data.tsx)
│
├── assets/                 ← Unchanged
├── styles/                 ← Unchanged
├── routes/AppRoutes.tsx    ← Updated imports only
├── App.tsx                 ← Updated to use new providers
└── main.tsx                ← Unchanged
```

**Removed after migration:**

- `src/contexts/` (entire directory)
- `src/base/` (entire directory)
- `src/components/Page/` (entire directory)
- `src/components/resuable/` (entire directory)
- `src/components/PlaceHolders/` (entire directory)
- `src/data/app_data.tsx`, `nav_data.tsx`, `data.tsx`, `service_data.tsx`

---

## Task 1: Install Redux Toolkit

**Files:**

- Modify: `package.json`

- [ ] **Step 1: Install dependencies**

```bash
npm install @reduxjs/toolkit react-redux
```

Expected output: `added X packages` with no errors.

- [ ] **Step 2: Verify build still passes**

```bash
npm run build
```

Expected: Build succeeds (no new errors introduced).

- [ ] **Step 3: Commit**

```bash
git add package.json package-lock.json
git commit -m "chore: add @reduxjs/toolkit and react-redux"
```

---

## Task 2: App store, typed hooks, and providers

**Files:**

- Create: `src/app/store.ts`
- Create: `src/app/hooks.ts`
- Create: `src/app/providers.tsx`

- [ ] **Step 1: Create `src/app/store.ts`**

```ts
import { configureStore } from '@reduxjs/toolkit';

// Slices are added here in Task 5 — store starts empty
export const store = configureStore({
  reducer: {},
});

export type RootState = ReturnType<typeof store.getState>;
export type AppDispatch = typeof store.dispatch;
```

- [ ] **Step 2: Create `src/app/hooks.ts`**

```ts
import { useDispatch, useSelector } from 'react-redux';
import type { RootState, AppDispatch } from './store';

export const useAppDispatch = useDispatch.withTypes<AppDispatch>();
export const useAppSelector = useSelector.withTypes<RootState>();
```

- [ ] **Step 3: Create `src/app/providers.tsx`**

This replaces `src/contexts/RootContext.tsx`. It keeps UI-only contexts (Theme, Device) and adds Redux `Provider`. BrowserRouter stays here.

```tsx
import React from 'react';
import { Provider } from 'react-redux';
import { BrowserRouter } from 'react-router-dom';
import { HelmetProvider } from 'react-helmet-async';
import { store } from './store';
import { ThemeProvider } from '../shared/hooks/useTheme';
import { DeviceProvider } from '../shared/hooks/useDevice';

type ProvidersProps = { children: React.ReactNode };

export const AppProviders: React.FC<ProvidersProps> = ({ children }) => (
  <Provider store={store}>
    <HelmetProvider>
      <ThemeProvider>
        <DeviceProvider>
          <BrowserRouter>{children}</BrowserRouter>
        </DeviceProvider>
      </ThemeProvider>
    </HelmetProvider>
  </Provider>
);
```

Note: `ThemeProvider` and `DeviceProvider` are created in Task 3.

- [ ] **Step 4: Verify TypeScript compiles**

```bash
npx tsc --noEmit
```

Expected: No errors in the three new files (may have unresolved import errors for files not yet created — acceptable at this stage).

- [ ] **Step 5: Commit**

```bash
git add src/app/
git commit -m "feat: add Redux store, typed hooks, and AppProviders"
```

---

## Task 3: Shared hooks — useTheme, useDevice, useScroll

These are extracted from `ThemeContext.tsx` and `DeviceContext.tsx`. They still use React Context internally (UI-only state is fine in Context) but are now in `shared/hooks/` and export a provider + hook.

**Files:**

- Create: `src/shared/hooks/useTheme.tsx`
- Create: `src/shared/hooks/useDevice.tsx`
- Create: `src/shared/hooks/useScroll.ts`

- [ ] **Step 1: Create `src/shared/hooks/useTheme.tsx`**

Copy logic from `src/contexts/ThemeContext.tsx` verbatim, update export names:

```tsx
import React, { createContext, useContext, useEffect, useState } from 'react';

type Theme = 'dark' | 'light';

interface ThemeContextType {
  theme: Theme;
  toggleTheme: () => void;
}

const ThemeContext = createContext<ThemeContextType>({
  theme: 'dark',
  toggleTheme: () => {},
});

export const ThemeProvider: React.FC<{ children: React.ReactNode }> = ({
  children,
}) => {
  const [theme, setTheme] = useState<Theme>(() => {
    const stored = localStorage.getItem('tl-theme');
    if (stored === 'light' || stored === 'dark') return stored;
    return 'dark';
  });

  useEffect(() => {
    const root = document.documentElement;
    if (theme === 'dark') {
      root.classList.add('dark');
    } else {
      root.classList.remove('dark');
    }
    localStorage.setItem('tl-theme', theme);
  }, [theme]);

  const toggleTheme = () => setTheme((t) => (t === 'dark' ? 'light' : 'dark'));

  return (
    <ThemeContext.Provider value={{ theme, toggleTheme }}>
      {children}
    </ThemeContext.Provider>
  );
};

export const useTheme = () => useContext(ThemeContext);
```

- [ ] **Step 2: Create `src/shared/hooks/useDevice.tsx`**

```tsx
import React, {
  createContext,
  useContext,
  useState,
  useLayoutEffect,
} from 'react';

export type DeviceType = 'xs' | 'sm' | 'md' | 'lg' | 'xl' | '2xl';

const DeviceContext = createContext<DeviceType>('lg');

const getDeviceType = (width: number): DeviceType => {
  if (width >= 1536) return '2xl';
  if (width >= 1280) return 'xl';
  if (width >= 1024) return 'lg';
  if (width >= 768) return 'md';
  if (width >= 640) return 'sm';
  return 'xs';
};

export const DeviceProvider: React.FC<{ children: React.ReactNode }> = ({
  children,
}) => {
  const [device, setDevice] = useState<DeviceType>('lg');

  useLayoutEffect(() => {
    const update = () => setDevice(getDeviceType(window.innerWidth));
    update();
    window.addEventListener('resize', update);
    return () => window.removeEventListener('resize', update);
  }, []);

  return (
    <DeviceContext.Provider value={device}>{children}</DeviceContext.Provider>
  );
};

export const useDevice = () => useContext(DeviceContext);
```

- [ ] **Step 3: Create `src/shared/hooks/useScroll.ts`**

Copy from `src/base/hooks/useScroll.tsx`:

```ts
import { useState, useEffect } from 'react';

export const useScroll = (threshold = 0) => {
  const [scrolled, setScrolled] = useState(false);

  useEffect(() => {
    const onScroll = () => setScrolled(window.scrollY > threshold);
    window.addEventListener('scroll', onScroll, { passive: true });
    return () => window.removeEventListener('scroll', onScroll);
  }, [threshold]);

  return scrolled;
};
```

(Read `src/base/hooks/useScroll.tsx` first to verify the exact implementation matches.)

- [ ] **Step 4: Commit**

```bash
git add src/shared/hooks/
git commit -m "feat: add shared hooks (useTheme, useDevice, useScroll)"
```

---

## Task 4: Shared types and utils

**Files:**

- Create: `src/shared/types/common.ts`
- Create: `src/shared/types/service.ts`
- Create: `src/shared/types/project.ts`
- Create: `src/shared/types/socialMedia.ts`
- Create: `src/shared/utils/throttle.ts`
- Create: `src/shared/utils/googleDrive.ts`

- [ ] **Step 1: Create `src/shared/types/common.ts`**

```ts
import { ReactNode } from 'react';

export type ParentType = {
  children?: ReactNode;
  className?: string;
  id?: string;
};

export type AnimateVariables = {
  initial?: object;
  animate?: object;
  exit?: object;
};

export type LinkProps = {
  content: ReactNode;
  link?: string;
  target?: string;
};

export type EventProps = {
  onClick?: (e: React.MouseEvent<HTMLDivElement>) => void;
  onHover?: (e: React.MouseEvent<HTMLDivElement>) => void;
  onDoubleClick?: (e: React.MouseEvent<HTMLDivElement>) => void;
  onTap?: (e: React.TouchEvent<HTMLDivElement>) => void;
  onScroll?: (e: React.UIEvent<HTMLDivElement>) => void;
};

export type ImageProps = {
  src: string;
  width?: number;
  height?: number;
  alt?: string;
  className?: string;
  loading: 'eager' | 'normal';
};
```

- [ ] **Step 2: Create `src/shared/types/service.ts`**

```ts
export type ServiceCardProps = {
  id: number;
  slug: string;
  service_name: string;
  description: string;
  icon: string;
  show_in_home?: string;
};
```

- [ ] **Step 3: Create `src/shared/types/project.ts`**

```ts
export type ProjectItem = {
  id: number;
  slug: string;
  title: string;
  imageURL: string;
  description: string;
  service: string;
  extraImages?: string[];
  content?: string;
  vedio_demo?: string;
  is_top?: boolean;
};
```

- [ ] **Step 4: Create `src/shared/types/socialMedia.ts`**

```ts
export type SocialMediaItem = {
  platform: string;
  url: string;
  icon?: string;
};
```

- [ ] **Step 5: Create `src/shared/utils/throttle.ts`**

Read `src/base/util/Throttle.tsx` and copy logic here:

```ts
export const throttle = <T extends (...args: unknown[]) => unknown>(
  fn: T,
  limit: number
): ((...args: Parameters<T>) => void) => {
  let lastCall = 0;
  return (...args: Parameters<T>) => {
    const now = Date.now();
    if (now - lastCall >= limit) {
      lastCall = now;
      fn(...args);
    }
  };
};
```

(Read `src/base/util/Throttle.tsx` to confirm the exact implementation before copying.)

- [ ] **Step 6: Create `src/shared/utils/googleDrive.ts`**

```ts
export function extractGoogleDriveFileId(url: string): string | null {
  const filePattern = /\/file\/d\/([a-zA-Z0-9_-]+)/;
  const openPattern = /[\?&]id=([a-zA-Z0-9_-]+)/;
  const docsPattern = /\/d\/([a-zA-Z0-9_-]+)/;

  return (
    url.match(filePattern)?.[1] ??
    url.match(openPattern)?.[1] ??
    url.match(docsPattern)?.[1] ??
    null
  );
}
```

- [ ] **Step 7: Commit**

```bash
git add src/shared/types/ src/shared/utils/
git commit -m "feat: add shared types and utils"
```

---

## Task 5: Service layer — pure fetch functions

These are plain TypeScript async functions. No React, no hooks, no state. Each one calls the Google Sheets Apps Script endpoint for one data domain.

**Files:**

- Create: `src/services/sheetsClient.ts`
- Create: `src/services/heroService.ts`
- Create: `src/services/servicesService.ts`
- Create: `src/services/projectsService.ts`
- Create: `src/services/aboutService.ts`
- Create: `src/services/contactService.ts`

- [ ] **Step 1: Create `src/services/sheetsClient.ts`**

```ts
const BASE_URL = import.meta.env.VITE_SHEET_URL as string;

export async function fetchSheet<T>(
  sheetName: string,
  signal?: AbortSignal
): Promise<T[]> {
  if (!BASE_URL) throw new Error('VITE_SHEET_URL is not defined');

  const response = await fetch(`${BASE_URL}${sheetName}`, { signal });
  if (!response.ok)
    throw new Error(`HTTP ${response.status} fetching ${sheetName}`);

  const result = await response.json();
  if (!Array.isArray(result?.data)) {
    throw new Error(`Unexpected response format from ${sheetName}`);
  }
  return result.data as T[];
}
```

- [ ] **Step 2: Create `src/services/heroService.ts`**

```ts
import { fetchSheet } from './sheetsClient';

export interface HeroSlide {
  id: string;
  imageURL: string;
  title?: string;
  subtitle?: string;
  [key: string]: unknown;
}

export const fetchHeroSlides = (signal?: AbortSignal): Promise<HeroSlide[]> =>
  fetchSheet<HeroSlide>('HeroData', signal);
```

- [ ] **Step 3: Create `src/services/servicesService.ts`**

```ts
import { fetchSheet } from './sheetsClient';
import type { ServiceCardProps } from '../shared/types/service';

export const fetchServices = (
  signal?: AbortSignal
): Promise<ServiceCardProps[]> =>
  fetchSheet<ServiceCardProps>('ServiceData', signal);
```

- [ ] **Step 4: Create `src/services/projectsService.ts`**

```ts
import { fetchSheet } from './sheetsClient';
import type { ProjectItem } from '../shared/types/project';

export const fetchProjects = (signal?: AbortSignal): Promise<ProjectItem[]> =>
  fetchSheet<ProjectItem>('ProjectData', signal);
```

- [ ] **Step 5: Create `src/services/aboutService.ts`**

```ts
import { fetchSheet } from './sheetsClient';

export interface AboutItem {
  [key: string]: unknown;
}

export const fetchAboutData = (signal?: AbortSignal): Promise<AboutItem[]> =>
  fetchSheet<AboutItem>('AboutData', signal);
```

- [ ] **Step 6: Create `src/services/contactService.ts`**

```ts
import { fetchSheet } from './sheetsClient';

export interface ContactItem {
  [key: string]: unknown;
}

export const fetchContactData = (
  signal?: AbortSignal
): Promise<ContactItem[]> => fetchSheet<ContactItem>('ContactData', signal);
```

- [ ] **Step 7: Commit**

```bash
git add src/services/
git commit -m "feat: add pure service layer for Google Sheets data fetching"
```

---

## Task 6: Redux slices

One slice per data domain. Each uses `createAsyncThunk` to call the service layer and `createSlice` to manage loading/error/data state.

**Files:**

- Create: `src/store/heroSlice.ts`
- Create: `src/store/servicesSlice.ts`
- Create: `src/store/projectsSlice.ts`
- Create: `src/store/aboutSlice.ts`
- Create: `src/store/contactSlice.ts`
- Modify: `src/app/store.ts`

- [ ] **Step 1: Create `src/store/heroSlice.ts`**

```ts
import { createSlice, createAsyncThunk, PayloadAction } from '@reduxjs/toolkit';
import { fetchHeroSlides, HeroSlide } from '../services/heroService';

interface HeroState {
  slides: HeroSlide[];
  currentIndex: number;
  status: 'idle' | 'loading' | 'succeeded' | 'failed';
  error: string | null;
}

const initialState: HeroState = {
  slides: [],
  currentIndex: 0,
  status: 'idle',
  error: null,
};

export const loadHeroSlides = createAsyncThunk(
  'hero/loadSlides',
  async (_, { signal }) => fetchHeroSlides(signal)
);

const normalise = (value: number, length: number) =>
  length > 0 ? ((value % length) + length) % length : 0;

export const heroSlice = createSlice({
  name: 'hero',
  initialState,
  reducers: {
    nextSlide(state) {
      state.currentIndex = normalise(
        state.currentIndex + 1,
        state.slides.length
      );
    },
    prevSlide(state) {
      state.currentIndex = normalise(
        state.currentIndex - 1,
        state.slides.length
      );
    },
    setSlide(state, action: PayloadAction<number>) {
      state.currentIndex = normalise(action.payload, state.slides.length);
    },
  },
  extraReducers: (builder) => {
    builder
      .addCase(loadHeroSlides.pending, (state) => {
        state.status = 'loading';
        state.error = null;
      })
      .addCase(loadHeroSlides.fulfilled, (state, action) => {
        state.status = 'succeeded';
        state.slides = action.payload;
        state.currentIndex = 0;
      })
      .addCase(loadHeroSlides.rejected, (state, action) => {
        state.status = 'failed';
        state.error = action.error.message ?? 'Failed to load hero slides';
      });
  },
});

export const { nextSlide, prevSlide, setSlide } = heroSlice.actions;
export default heroSlice.reducer;

// Selectors
export const selectHeroSlides = (state: { hero: HeroState }) =>
  state.hero.slides;
export const selectCurrentSlide = (state: { hero: HeroState }) =>
  state.hero.slides[state.hero.currentIndex] ?? null;
export const selectCurrentIndex = (state: { hero: HeroState }) =>
  state.hero.currentIndex;
export const selectHeroStatus = (state: { hero: HeroState }) =>
  state.hero.status;
```

- [ ] **Step 2: Create `src/store/servicesSlice.ts`**

```ts
import { createSlice, createAsyncThunk } from '@reduxjs/toolkit';
import { fetchServices } from '../services/servicesService';
import type { ServiceCardProps } from '../shared/types/service';

interface ServicesState {
  items: ServiceCardProps[];
  status: 'idle' | 'loading' | 'succeeded' | 'failed';
  error: string | null;
}

const initialState: ServicesState = {
  items: [],
  status: 'idle',
  error: null,
};

export const loadServices = createAsyncThunk(
  'services/load',
  async (_, { signal }) => fetchServices(signal)
);

export const servicesSlice = createSlice({
  name: 'services',
  initialState,
  reducers: {},
  extraReducers: (builder) => {
    builder
      .addCase(loadServices.pending, (state) => {
        state.status = 'loading';
        state.error = null;
      })
      .addCase(loadServices.fulfilled, (state, action) => {
        state.status = 'succeeded';
        state.items = action.payload;
      })
      .addCase(loadServices.rejected, (state, action) => {
        state.status = 'failed';
        state.error = action.error.message ?? 'Failed to load services';
      });
  },
});

export default servicesSlice.reducer;

export const selectServices = (state: { services: ServicesState }) =>
  state.services.items;
export const selectServicesStatus = (state: { services: ServicesState }) =>
  state.services.status;
export const selectHomeServices = (state: { services: ServicesState }) =>
  state.services.items.filter(
    (s) => s.show_in_home === 'true' || s.show_in_home === '1'
  );
```

- [ ] **Step 3: Create `src/store/projectsSlice.ts`**

```ts
import { createSlice, createAsyncThunk } from '@reduxjs/toolkit';
import { fetchProjects } from '../services/projectsService';
import type { ProjectItem } from '../shared/types/project';

interface ProjectsState {
  items: ProjectItem[];
  status: 'idle' | 'loading' | 'succeeded' | 'failed';
  error: string | null;
  activeServiceSlug: string;
  currentPage: number;
}

const initialState: ProjectsState = {
  items: [],
  status: 'idle',
  error: null,
  activeServiceSlug: 'all',
  currentPage: 1,
};

export const loadProjects = createAsyncThunk(
  'projects/load',
  async (_, { signal }) => fetchProjects(signal)
);

export const projectsSlice = createSlice({
  name: 'projects',
  initialState,
  reducers: {
    setActiveServiceSlug(state, action: { payload: string }) {
      state.activeServiceSlug = action.payload;
      state.currentPage = 1;
    },
    setCurrentPage(state, action: { payload: number }) {
      state.currentPage = action.payload;
    },
  },
  extraReducers: (builder) => {
    builder
      .addCase(loadProjects.pending, (state) => {
        state.status = 'loading';
        state.error = null;
      })
      .addCase(loadProjects.fulfilled, (state, action) => {
        state.status = 'succeeded';
        state.items = action.payload;
      })
      .addCase(loadProjects.rejected, (state, action) => {
        state.status = 'failed';
        state.error = action.error.message ?? 'Failed to load projects';
      });
  },
});

export const { setActiveServiceSlug, setCurrentPage } = projectsSlice.actions;
export default projectsSlice.reducer;

export const selectAllProjects = (state: { projects: ProjectsState }) =>
  state.projects.items;
export const selectProjectsStatus = (state: { projects: ProjectsState }) =>
  state.projects.status;
export const selectActiveSlug = (state: { projects: ProjectsState }) =>
  state.projects.activeServiceSlug;
export const selectCurrentPage = (state: { projects: ProjectsState }) =>
  state.projects.currentPage;
export const selectTopProjects = (state: { projects: ProjectsState }) =>
  state.projects.items.filter((p) => p.is_top);
export const selectProjectBySlug =
  (slug: string) => (state: { projects: ProjectsState }) =>
    state.projects.items.find((p) => p.slug === slug) ?? null;
```

- [ ] **Step 4: Create `src/store/aboutSlice.ts`**

```ts
import { createSlice, createAsyncThunk } from '@reduxjs/toolkit';
import { fetchAboutData, AboutItem } from '../services/aboutService';

interface AboutState {
  data: AboutItem[];
  status: 'idle' | 'loading' | 'succeeded' | 'failed';
  error: string | null;
}

const initialState: AboutState = { data: [], status: 'idle', error: null };

export const loadAboutData = createAsyncThunk(
  'about/load',
  async (_, { signal }) => fetchAboutData(signal)
);

export const aboutSlice = createSlice({
  name: 'about',
  initialState,
  reducers: {},
  extraReducers: (builder) => {
    builder
      .addCase(loadAboutData.pending, (state) => {
        state.status = 'loading';
        state.error = null;
      })
      .addCase(loadAboutData.fulfilled, (state, action) => {
        state.status = 'succeeded';
        state.data = action.payload;
      })
      .addCase(loadAboutData.rejected, (state, action) => {
        state.status = 'failed';
        state.error = action.error.message ?? 'Failed to load about data';
      });
  },
});

export default aboutSlice.reducer;
export const selectAboutData = (state: { about: AboutState }) =>
  state.about.data;
export const selectAboutStatus = (state: { about: AboutState }) =>
  state.about.status;
```

- [ ] **Step 5: Create `src/store/contactSlice.ts`**

```ts
import { createSlice, createAsyncThunk } from '@reduxjs/toolkit';
import { fetchContactData, ContactItem } from '../services/contactService';

interface ContactState {
  data: ContactItem[];
  status: 'idle' | 'loading' | 'succeeded' | 'failed';
  error: string | null;
}

const initialState: ContactState = { data: [], status: 'idle', error: null };

export const loadContactData = createAsyncThunk(
  'contact/load',
  async (_, { signal }) => fetchContactData(signal)
);

export const contactSlice = createSlice({
  name: 'contact',
  initialState,
  reducers: {},
  extraReducers: (builder) => {
    builder
      .addCase(loadContactData.pending, (state) => {
        state.status = 'loading';
        state.error = null;
      })
      .addCase(loadContactData.fulfilled, (state, action) => {
        state.status = 'succeeded';
        state.data = action.payload;
      })
      .addCase(loadContactData.rejected, (state, action) => {
        state.status = 'failed';
        state.error = action.error.message ?? 'Failed to load contact data';
      });
  },
});

export default contactSlice.reducer;
export const selectContactData = (state: { contact: ContactState }) =>
  state.contact.data;
export const selectContactStatus = (state: { contact: ContactState }) =>
  state.contact.status;
```

- [ ] **Step 6: Register slices in `src/app/store.ts`**

```ts
import { configureStore } from '@reduxjs/toolkit';
import heroReducer from '../store/heroSlice';
import servicesReducer from '../store/servicesSlice';
import projectsReducer from '../store/projectsSlice';
import aboutReducer from '../store/aboutSlice';
import contactReducer from '../store/contactSlice';

export const store = configureStore({
  reducer: {
    hero: heroReducer,
    services: servicesReducer,
    projects: projectsReducer,
    about: aboutReducer,
    contact: contactReducer,
  },
});

export type RootState = ReturnType<typeof store.getState>;
export type AppDispatch = typeof store.dispatch;
```

- [ ] **Step 7: Type-check**

```bash
npx tsc --noEmit
```

Expected: No errors in `src/app/` or `src/store/`.

- [ ] **Step 8: Commit**

```bash
git add src/store/ src/app/store.ts
git commit -m "feat: add Redux slices for hero, services, projects, about, contact"
```

---

## Task 7: Shared UI components

Move `components/resuable/`, `components/PlaceHolders/`, and `components/three/` to `shared/components/`.

**Files:**

- Create: `src/shared/components/ui/Card.tsx`
- Create: `src/shared/components/ui/Section.tsx`
- Create: `src/shared/components/ui/PageWrapper.tsx`
- Create: `src/shared/components/feedback/ErrorBoundary.tsx`
- Create: `src/shared/components/feedback/OfflineWarning.tsx`
- Create: `src/shared/components/three/GlobalBackground.tsx`

- [ ] **Step 1: Copy `Card.tsx`**

Read `src/components/resuable/Card.tsx`. Create `src/shared/components/ui/Card.tsx` with identical content, updating any imports that used old paths.

- [ ] **Step 2: Copy `Section.tsx`**

Read `src/components/resuable/Section.tsx`. Create `src/shared/components/ui/Section.tsx` with identical content.

- [ ] **Step 3: Copy `Page.tsx` → `PageWrapper.tsx`**

Read `src/components/resuable/Page.tsx`. Create `src/shared/components/ui/PageWrapper.tsx` with identical content, renaming the component from `Page` to `PageWrapper` for clarity.

- [ ] **Step 4: Copy `ErrorBoundary.tsx`**

Read `src/components/PlaceHolders/ErrorBoundary.tsx`. Create `src/shared/components/feedback/ErrorBoundary.tsx` with identical content.

- [ ] **Step 5: Copy `OfflineWarning.tsx`**

Read `src/components/PlaceHolders/OfflineWarning.tsx`. Create `src/shared/components/feedback/OfflineWarning.tsx` with identical content.

- [ ] **Step 6: Copy `GlobalBackground.tsx`**

Read `src/components/three/GlobalBackground.tsx`. Create `src/shared/components/three/GlobalBackground.tsx` with identical content.

- [ ] **Step 7: Commit**

```bash
git add src/shared/components/
git commit -m "feat: move reusable UI, feedback, and three components to shared"
```

---

## Task 8: Shared layout components

Move `components/layout/` to `shared/components/layout/` and update internal imports.

**Files:**

- Create: `src/shared/components/layout/Header.tsx`
- Create: `src/shared/components/layout/Footer.tsx`
- Create: `src/shared/components/layout/NavBar.tsx`
- Create: `src/shared/components/layout/NavItem.tsx`
- Create: `src/shared/components/layout/MobileNavigation.tsx`
- Create: `src/shared/components/layout/Layout.tsx`

- [ ] **Step 1: For each layout file**

Read the original in `src/components/layout/`. Create the copy in `src/shared/components/layout/`. Update internal imports:

- `useDeviceContext` → `useDevice` from `../../hooks/useDevice`
- `useTheme` from `../../hooks/useTheme`
- `useScroll` from `../../hooks/useScroll`
- Nav data from `../../../data/nav`

Do this for: `Header.tsx`, `Footer.tsx`, `NavBar.tsx`, `NavItem.tsx`, `MobileNavigation.tsx`, `Layout.tsx`.

- [ ] **Step 2: Type-check after layout migration**

```bash
npx tsc --noEmit
```

- [ ] **Step 3: Commit**

```bash
git add src/shared/components/layout/
git commit -m "feat: move layout components to shared"
```

---

## Task 9: Feature — hero

**Files:**

- Create: `src/features/hero/hooks/useHeroController.ts`
- Create: `src/features/hero/components/HeroSection.tsx`
- Create: `src/features/hero/components/HeroImageSlider.tsx`
- Create: `src/features/hero/components/HeroKeyPoint.tsx`
- Create: `src/features/hero/components/HeroLoading.tsx`

- [ ] **Step 1: Create `src/features/hero/hooks/useHeroController.ts`**

This replaces `HeroContext.tsx`. It dispatches thunks, manages auto-advance interval, and exposes clean data to the view.

```ts
import { useEffect } from 'react';
import { useAppDispatch, useAppSelector } from '../../../app/hooks';
import {
  loadHeroSlides,
  nextSlide,
  prevSlide,
  setSlide,
  selectCurrentSlide,
  selectCurrentIndex,
  selectHeroSlides,
  selectHeroStatus,
} from '../../../store/heroSlice';

const SLIDE_INTERVAL_MS = 5000;

export const useHeroController = () => {
  const dispatch = useAppDispatch();
  const slides = useAppSelector(selectHeroSlides);
  const currentSlide = useAppSelector(selectCurrentSlide);
  const currentIndex = useAppSelector(selectCurrentIndex);
  const status = useAppSelector(selectHeroStatus);

  useEffect(() => {
    const controller = new AbortController();
    dispatch(loadHeroSlides());
    return () => controller.abort();
  }, [dispatch]);

  useEffect(() => {
    if (slides.length <= 1) return;
    const interval = setInterval(
      () => dispatch(nextSlide()),
      SLIDE_INTERVAL_MS
    );
    return () => clearInterval(interval);
  }, [dispatch, slides.length]);

  return {
    slides,
    currentSlide,
    currentIndex,
    isLoading: status === 'loading' || status === 'idle',
    goNext: () => dispatch(nextSlide()),
    goPrev: () => dispatch(prevSlide()),
    goTo: (i: number) => dispatch(setSlide(i)),
  };
};
```

- [ ] **Step 2: Migrate hero components**

Read each file in `src/components/Page/Home/Hero/`. Create copies under `src/features/hero/components/`. Replace:

- `useHeroContext()` with the return value of `useHeroController()` (passed as props or called at the top level)
- `useDeviceContext` → `useDevice` from `../../../shared/hooks/useDevice`
- `Card`, `Section` → from `../../../shared/components/ui/`

- [ ] **Step 3: Commit**

```bash
git add src/features/hero/
git commit -m "feat: migrate hero feature to Redux + controller hook"
```

---

## Task 10: Feature — home-services

**Files:**

- Create: `src/features/home-services/hooks/useHomeServicesController.ts`
- Create: `src/features/home-services/components/ServiceSection.tsx`
- Create: `src/features/home-services/components/ServiceCard.tsx`

- [ ] **Step 1: Create `src/features/home-services/hooks/useHomeServicesController.ts`**

```ts
import { useEffect } from 'react';
import { useAppDispatch, useAppSelector } from '../../../app/hooks';
import {
  loadServices,
  selectHomeServices,
  selectServicesStatus,
} from '../../../store/servicesSlice';

export const useHomeServicesController = () => {
  const dispatch = useAppDispatch();
  const services = useAppSelector(selectHomeServices);
  const status = useAppSelector(selectServicesStatus);

  useEffect(() => {
    if (status === 'idle') dispatch(loadServices());
  }, [dispatch, status]);

  return {
    services,
    isLoading: status === 'loading' || status === 'idle',
  };
};
```

- [ ] **Step 2: Migrate service components**

Read `src/components/Page/Home/Service/ServiceSection.tsx` and `ServiceCard.tsx`. Create copies under `src/features/home-services/components/`. Replace `useServiceDataContext` with the controller return value.

- [ ] **Step 3: Commit**

```bash
git add src/features/home-services/
git commit -m "feat: migrate home-services feature to Redux + controller hook"
```

---

## Task 11: Feature — home-projects

**Files:**

- Create: `src/features/home-projects/hooks/useLatestProjectsController.ts`
- Create: `src/features/home-projects/components/` (5 components from `LatestPro/`)

- [ ] **Step 1: Create `src/features/home-projects/hooks/useLatestProjectsController.ts`**

```ts
import { useEffect } from 'react';
import { useAppDispatch, useAppSelector } from '../../../app/hooks';
import {
  loadProjects,
  selectTopProjects,
  selectProjectsStatus,
} from '../../../store/projectsSlice';

const AUTO_ROTATE_MS = 8000;

export const useLatestProjectsController = () => {
  const dispatch = useAppDispatch();
  const projects = useAppSelector(selectTopProjects);
  const status = useAppSelector(selectProjectsStatus);

  useEffect(() => {
    if (status === 'idle') dispatch(loadProjects());
  }, [dispatch, status]);

  return {
    projects,
    isLoading: status === 'loading' || status === 'idle',
    autoRotateMs: AUTO_ROTATE_MS,
  };
};
```

- [ ] **Step 2: Migrate LatestPro components**

Read each file in `src/components/Page/Home/LatestPro/`. Create copies under `src/features/home-projects/components/`. Replace context imports with controller return values and shared imports.

- [ ] **Step 3: Commit**

```bash
git add src/features/home-projects/
git commit -m "feat: migrate home-projects feature to Redux + controller hook"
```

---

## Task 12: Feature — services-page

**Files:**

- Create: `src/features/services-page/hooks/useServicesPageController.ts`
- Create: `src/features/services-page/components/` (8 components from `Service/`)

- [ ] **Step 1: Create `src/features/services-page/hooks/useServicesPageController.ts`**

```ts
import { useEffect } from 'react';
import { useParams, useLocation, useNavigate } from 'react-router-dom';
import { useAppDispatch, useAppSelector } from '../../../app/hooks';
import {
  loadProjects, loadServices (if not already loaded),
  setActiveServiceSlug, setCurrentPage,
  selectAllProjects, selectProjectsStatus,
  selectActiveSlug, selectCurrentPage,
} from '../../../store/projectsSlice';
import { selectServices } from '../../../store/servicesSlice';

const PER_PAGE = 6;

export const useServicesPageController = () => {
  const dispatch = useAppDispatch();
  const { slug } = useParams<{ slug: string }>();
  const location = useLocation();
  const navigate = useNavigate();

  const allProjects = useAppSelector(selectAllProjects);
  const status = useAppSelector(selectProjectsStatus);
  const activeSlug = useAppSelector(selectActiveSlug);
  const currentPage = useAppSelector(selectCurrentPage);
  const services = useAppSelector(selectServices);

  useEffect(() => {
    if (status === 'idle') dispatch(loadProjects());
  }, [dispatch, status]);

  useEffect(() => {
    if (slug) dispatch(setActiveServiceSlug(slug));
  }, [dispatch, slug]);

  useEffect(() => {
    const page = parseInt(new URLSearchParams(location.search).get('page') ?? '1', 10) || 1;
    dispatch(setCurrentPage(page));
  }, [dispatch, location.search]);

  const filtered = activeSlug === 'all'
    ? allProjects
    : allProjects.filter(p => p.service === activeSlug);

  const totalItems = filtered.length;
  const paged = filtered.slice((currentPage - 1) * PER_PAGE, currentPage * PER_PAGE);

  const setTab = (tabSlug: string) => {
    navigate(`/services/${tabSlug}`);
  };

  return {
    projects: paged,
    totalItems,
    currentPage,
    perPage: PER_PAGE,
    services,
    activeSlug,
    isLoading: status === 'loading' || status === 'idle',
    setTab,
  };
};
```

- [ ] **Step 2: Migrate Service page components**

Read each file in `src/components/Page/Service/`. Create copies under `src/features/services-page/components/`. Replace context calls with controller return values.

- [ ] **Step 3: Commit**

```bash
git add src/features/services-page/
git commit -m "feat: migrate services-page feature to Redux + controller hook"
```

---

## Task 13: Feature — project-detail

**Files:**

- Create: `src/features/project-detail/hooks/useProjectDetailController.ts`
- Create: `src/features/project-detail/components/ProjectHero.tsx`
- Create: `src/features/project-detail/components/ProjectNotFound.tsx`
- Create: `src/features/project-detail/components/ProjectPageLoading.tsx`

- [ ] **Step 1: Create `src/features/project-detail/hooks/useProjectDetailController.ts`**

```ts
import { useEffect } from 'react';
import { useParams } from 'react-router-dom';
import { useAppDispatch, useAppSelector } from '../../../app/hooks';
import {
  loadProjects,
  selectProjectBySlug,
  selectProjectsStatus,
} from '../../../store/projectsSlice';

export const useProjectDetailController = () => {
  const dispatch = useAppDispatch();
  const { slug } = useParams<{ slug: string }>();
  const status = useAppSelector(selectProjectsStatus);
  const project = useAppSelector(selectProjectBySlug(slug ?? ''));

  useEffect(() => {
    if (status === 'idle') dispatch(loadProjects());
  }, [dispatch, status]);

  return {
    project,
    isLoading: status === 'loading' || status === 'idle',
    notFound: status === 'succeeded' && !project,
  };
};
```

- [ ] **Step 2: Migrate project detail components**

Read `src/components/Page/Project/ProjectHero.tsx`, `ProjectNotFound.tsx`, `ProjectPageLoading.tsx`. Create copies under `src/features/project-detail/components/`.

- [ ] **Step 3: Commit**

```bash
git add src/features/project-detail/
git commit -m "feat: migrate project-detail feature to Redux + controller hook"
```

---

## Task 14: Features — about and contact

**Files:**

- Create: `src/features/about/hooks/useAboutController.ts`
- Create: `src/features/about/components/AboutUsSection.tsx`
- Create: `src/features/about/components/AboutLoading.tsx`
- Create: `src/features/contact/hooks/useContactController.ts`
- Create: `src/features/contact/components/ContactInfo.tsx`
- Create: `src/features/contact/components/SocialMediaLinks.tsx`
- Create: `src/features/contact/components/ContactUsPlaceholder.tsx`

- [ ] **Step 1: Create `src/features/about/hooks/useAboutController.ts`**

```ts
import { useEffect } from 'react';
import { useAppDispatch, useAppSelector } from '../../../app/hooks';
import {
  loadAboutData,
  selectAboutData,
  selectAboutStatus,
} from '../../../store/aboutSlice';

export const useAboutController = () => {
  const dispatch = useAppDispatch();
  const data = useAppSelector(selectAboutData);
  const status = useAppSelector(selectAboutStatus);

  useEffect(() => {
    if (status === 'idle') dispatch(loadAboutData());
  }, [dispatch, status]);

  return { data, isLoading: status === 'loading' || status === 'idle' };
};
```

- [ ] **Step 2: Create `src/features/contact/hooks/useContactController.ts`**

```ts
import { useEffect } from 'react';
import { useAppDispatch, useAppSelector } from '../../../app/hooks';
import {
  loadContactData,
  selectContactData,
  selectContactStatus,
} from '../../../store/contactSlice';

export const useContactController = () => {
  const dispatch = useAppDispatch();
  const data = useAppSelector(selectContactData);
  const status = useAppSelector(selectContactStatus);

  useEffect(() => {
    if (status === 'idle') dispatch(loadContactData());
  }, [dispatch, status]);

  return { data, isLoading: status === 'loading' || status === 'idle' };
};
```

- [ ] **Step 3: Migrate about and contact components**

Read each component in `src/components/Page/Aboutus/` and `src/components/Page/Contactus/`. Create copies in the respective feature directories. Replace context calls with controller returns.

- [ ] **Step 4: Commit**

```bash
git add src/features/about/ src/features/contact/
git commit -m "feat: migrate about and contact features to Redux + controller hooks"
```

---

## Task 15: Update pages (thin wrappers)

Pages become thin assemblers — they call the feature components and nothing else.

**Files:**

- Modify: `src/pages/Home.tsx`
- Modify: `src/pages/AboutUs.tsx`
- Modify: `src/pages/ContactUs.tsx`
- Modify: `src/pages/Services.tsx`
- Modify: `src/pages/Project.tsx`

- [ ] **Step 1: Rewrite `src/pages/Home.tsx`**

```tsx
import HeroSection from '../features/hero/components/HeroSection';
import ServiceSection from '../features/home-services/components/ServiceSection';
import LatestProductSection from '../features/home-projects/components/LatestProductSection';
import PageWrapper from '../shared/components/ui/PageWrapper';

export default function HomePage() {
  return (
    <PageWrapper
      title="Home | Tensor Labs"
      description="Engineering & Technology Solutions"
    >
      <HeroSection />
      <ServiceSection />
      <LatestProductSection />
    </PageWrapper>
  );
}
```

- [ ] **Step 2: Rewrite `src/pages/Services.tsx`**

```tsx
import ServiceHero from '../features/services-page/components/ServiceHero';
import ServiceContainer from '../features/services-page/components/ServiceContainer';
import PageWrapper from '../shared/components/ui/PageWrapper';

export default function ServicesPage() {
  return (
    <PageWrapper
      title="Services | Tensor Labs"
      description="Our engineering services"
    >
      <ServiceHero />
      <ServiceContainer />
    </PageWrapper>
  );
}
```

- [ ] **Step 3: Rewrite `src/pages/Project.tsx`**

```tsx
import ProjectHero from '../features/project-detail/components/ProjectHero';
import PageWrapper from '../shared/components/ui/PageWrapper';

export default function ProjectPage() {
  return (
    <PageWrapper title="Project | Tensor Labs" description="Project detail">
      <ProjectHero />
    </PageWrapper>
  );
}
```

- [ ] **Step 4: Rewrite `src/pages/AboutUs.tsx` and `src/pages/ContactUs.tsx`**

Same pattern: PageWrapper + feature component. Read the existing files to confirm current structure before rewriting.

- [ ] **Step 5: Commit**

```bash
git add src/pages/
git commit -m "refactor: pages become thin feature assemblers"
```

---

## Task 16: Update App.tsx and routes, wire providers

**Files:**

- Modify: `src/App.tsx`
- Modify: `src/routes/AppRoutes.tsx`
- Modify: `src/main.tsx` (if needed)

- [ ] **Step 1: Rewrite `src/App.tsx`**

```tsx
import React, { memo } from 'react';
import { AppProviders } from './app/providers';
import Layout from './shared/components/layout/Layout';
import ErrorBoundary from './shared/components/feedback/ErrorBoundary';
import OfflineWarning from './shared/components/feedback/OfflineWarning';

const App: React.FC = memo(() => (
  <ErrorBoundary>
    <AppProviders>
      <div className="min-h-screen flex flex-col">
        <OfflineWarning />
        <Layout />
      </div>
    </AppProviders>
  </ErrorBoundary>
));

App.displayName = 'App';
export default App;
```

- [ ] **Step 2: Update `src/routes/AppRoutes.tsx`**

Update page imports to use the new filenames (same route structure):

```tsx
import { Route, Routes, Navigate } from 'react-router-dom';
import HomePage from '../pages/Home';
import AboutPage from '../pages/AboutUs';
import ContactPage from '../pages/ContactUs';
import ServicesPage from '../pages/Services';
import ProjectPage from '../pages/Project';

export default function AppRoutes() {
  return (
    <Routes>
      <Route path="/" element={<HomePage />} />
      <Route path="/about-us" element={<AboutPage />} />
      <Route path="/contact-us" element={<ContactPage />} />
      <Route
        path="/services"
        element={<Navigate to="/services/all" replace />}
      />
      <Route path="/services/:slug" element={<ServicesPage />} />
      <Route path="/project/:slug" element={<ProjectPage />} />
    </Routes>
  );
}
```

- [ ] **Step 3: Full build check**

```bash
npm run build
```

Expected: Build succeeds with no TypeScript or Vite errors.

- [ ] **Step 4: Commit**

```bash
git add src/App.tsx src/routes/
git commit -m "refactor: wire AppProviders and update App entry point"
```

---

## Task 17: Rename static data files

**Files:**

- Create: `src/data/app.ts` ← from `app_data.tsx`
- Create: `src/data/nav.ts` ← from `nav_data.tsx`
- Create: `src/data/static.ts` ← from `data.tsx`

- [ ] **Step 1: For each data file**

Read the original. Create a copy at the new path with `.ts` extension (or `.tsx` if JSX is used). Update all imports that reference the old paths.

- [ ] **Step 2: Update all imports**

Search for usages of the old data files:

```bash
grep -r "from.*app_data\|from.*nav_data\|from.*data/data\|from.*service_data" src/ --include="*.tsx" --include="*.ts"
```

Update each import to the new path.

- [ ] **Step 3: Commit**

```bash
git add src/data/
git commit -m "refactor: rename static data files to clean names"
```

---

## Task 18: Remove old code

Only run this task after `npm run build` passes cleanly in Task 16.

**Files to delete:**

- `src/contexts/` (entire directory)
- `src/base/` (entire directory)
- `src/components/Page/` (entire directory)
- `src/components/resuable/` (entire directory)
- `src/components/PlaceHolders/` (entire directory)
- `src/components/layout/` (entire directory)
- `src/components/three/` (entire directory)
- `src/data/app_data.tsx`, `nav_data.tsx`, `data.tsx`, `service_data.tsx`

- [ ] **Step 1: Verify no imports to old paths remain**

```bash
grep -r "from.*contexts/\|from.*base/\|from.*components/Page/\|from.*components/resuable/\|from.*components/PlaceHolders/\|from.*components/layout/\|from.*components/three/" src/ --include="*.tsx" --include="*.ts"
```

Expected: zero matches.

- [ ] **Step 2: Delete old directories**

```bash
rm -rf src/contexts src/base src/components/Page src/components/resuable src/components/PlaceHolders src/components/layout src/components/three
rm -f src/data/app_data.tsx src/data/nav_data.tsx src/data/data.tsx src/data/service_data.tsx
```

- [ ] **Step 3: Final build + type check**

```bash
npx tsc --noEmit && npm run build
```

Expected: Both pass cleanly.

- [ ] **Step 4: Final commit**

```bash
git add -A
git commit -m "refactor: remove legacy contexts, base, and page component directories"
```

---

## Self-Review

### Spec coverage

| Requirement                                    | Covered in    |
| ---------------------------------------------- | ------------- |
| Pure service layer                             | Task 5        |
| Redux for remote state                         | Tasks 6, 9–14 |
| Controller hooks per feature                   | Tasks 9–14    |
| UI components decoupled from fetch logic       | Tasks 9–14    |
| Shared types/utils                             | Task 4        |
| Shared layout components                       | Task 8        |
| Context kept only for UI state (theme, device) | Task 3        |
| Pages as thin wrappers                         | Task 15       |
| Old code removed                               | Task 18       |

### Key notes for implementer

1. **Import aliasing:** Vite supports path aliases. Consider adding `@/` → `src/` in `vite.config.ts` to avoid `../../../` chains.
2. **`loadServices` deduplication:** Multiple features call `loadServices` — the `status === 'idle'` guard in each controller hook prevents duplicate fetches.
3. **`ServiceContext` (tab navigation):** The active service slug is now in `projectsSlice` (`activeServiceSlug`). The `ServiceContext` and `useNavigate`-based side effect is replaced by `useServicesPageController` calling `navigate()` directly on tab selection.
4. **`HeroContext` auto-advance:** Moved into `useHeroController` with a standard `setInterval`.
5. **`SocialMediaContext`:** Social links are loaded as part of `ContactData`. Merge into `contactSlice` during Task 14.
