
This document outlines a comprehensive implementation of internationalization (i18n) for protected routes in a Next.js application using Clerk for authentication and Convex as a backend.

## Overview

The implementation focuses on:

- Creating a language-specific URL structure for protected routes (`/(protected)/[lang]/route`)
- Maintaining Clerk authentication flow
- Using React Context for translation management
- Providing a language switcher component
- Handling default language redirection

## Project Structure

```
app/
├── (protected)/
│   ├── page.tsx                # Default redirect to English
│   └── [lang]/                 # Language parameter
│       ├── layout.tsx          # Layout with sidebar
│       ├── contract-checker/   # Protected page example
│       │   └── page.tsx
│       └── legal-ai/           # Protected page example
│           └── page.tsx
├── consts/
│   └── translations/           # Translation files
│       ├── en.ts
│       ├── fr.ts
│       └── es.ts
├── components/
│   ├── LanguageSwitcher.tsx    # Language selector component
│   └── LanguageSynchronizer.tsx # Client component for syncing URL language
├── context/
│   └── LanguageContext.tsx     # Translation context provider
└── middleware.ts               # Combined Clerk + language middleware
```

## Key Components

### 1. Translation Files

```typescript
// consts/translations/en.ts
export default {
  protected: {
    common: {
      dashboard: 'Dashboard',
      settings: 'Settings',
      logout: 'Logout',
    },
    contractChecker: {
      title: 'Contract Checker',
      description: 'Analyze your contracts for potential issues.',
      uploadButton: 'Upload Contract',
    },
    legalAI: {
      title: 'Legal AI Assistant',
      description: 'Get AI-powered legal advice and document analysis.',
      askQuestion: 'Ask a legal question',
    }
  }
};
```

Create similar files for `fr.ts` and `es.ts` with translated content.

### 2. Language Context

```typescript
// context/LanguageContext.tsx
'use client';

import { createContext, useState, useContext, useEffect, ReactNode } from 'react';

// Import all translation files
import en from '@/consts/translations/en';
import fr from '@/consts/translations/fr';
import es from '@/consts/translations/es';

// Define types for the translations
type TranslationDictionary = Record<string, any>;

// Define the available languages
type AvailableLanguage = 'en' | 'fr' | 'es';

const translations: Record<AvailableLanguage, TranslationDictionary> = {
  en,
  fr,
  es
};

// Define the context type
interface LanguageContextType {
  lang: AvailableLanguage;
  changeLang: (newLang: AvailableLanguage) => void;
  t: (key: string) => string;
}

// Create the context with a default value
const LanguageContext = createContext<LanguageContextType>({
  lang: 'en',
  changeLang: () => {},
  t: (key: string) => key
});

interface LanguageProviderProps {
  children: ReactNode;
  initialLang?: AvailableLanguage;
}

export function LanguageProvider({ 
  children, 
  initialLang = 'en' 
}: LanguageProviderProps) {
  const [lang, setLang] = useState<AvailableLanguage>(initialLang);
  
  // Load language preference from localStorage (client-side only)
  useEffect(() => {
    if (typeof window !== 'undefined') {
      const savedLang = localStorage.getItem('preferredLanguage');
      if (savedLang && ['en', 'fr', 'es'].includes(savedLang)) {
        setLang(savedLang as AvailableLanguage);
      }
    }
  }, []);
  
  // Save language preference when it changes
  useEffect(() => {
    if (typeof window !== 'undefined') {
      localStorage.setItem('preferredLanguage', lang);
    }
  }, [lang]);
  
  const changeLang = (newLang: AvailableLanguage): void => {
    if (['en', 'fr', 'es'].includes(newLang)) {
      setLang(newLang);
    }
  };
  
  // Translation function
  const t = (key: string): string => {
    const keys = key.split('.');
    let result: any = translations[lang] || translations.en;
    
    for (const k of keys) {
      if (result && result[k] !== undefined) {
        result = result[k];
      } else {
        console.warn(`Translation key not found: ${key}`);
        return key;
      }
    }
    
    return result as string;
  };
  
  return (
    <LanguageContext.Provider 
      value={{ 
        lang, 
        changeLang, 
        t 
      }}
    >
      {children}
    </LanguageContext.Provider>
  );
}

export function useLanguage(): LanguageContextType {
  const context = useContext(LanguageContext);
  return context;
}
```

### 3. Language Switcher Component

```typescript
// components/LanguageSwitcher.tsx
'use client';

import { useLanguage } from '@/context/LanguageContext';

type AvailableLanguage = 'en' | 'fr' | 'es';

interface Language {
  code: AvailableLanguage;
  name: string;
}

export default function LanguageSwitcher() {
  const { lang, changeLang } = useLanguage();
  
  const languages: Language[] = [
    { code: 'en', name: 'English' },
    { code: 'fr', name: 'Français' },
    { code: 'es', name: 'Español' }
  ];
  
  return (
    <div className="language-switcher">
      {languages.map((language) => (
        <button
          key={language.code}
          onClick={() => changeLang(language.code)}
          className={`lang-button ${lang === language.code ? 'active' : ''}`}
          aria-label={`Switch language to ${language.name}`}
        >
          {language.name}
        </button>
      ))}
    </div>
  );
}
```

### 4. Language Synchronizer (Client Component)

```typescript
// components/LanguageSynchronizer.tsx
'use client';

import { useEffect } from 'react';
import { useParams } from 'next/navigation';
import { useLanguage } from '@/context/LanguageContext';

type AvailableLanguage = 'en' | 'fr' | 'es';
const availableLanguages: AvailableLanguage[] = ['en', 'fr', 'es'];

export default function LanguageSynchronizer() {
  const params = useParams();
  const rawUrlLang = params.lang;
  const urlLang = Array.isArray(rawUrlLang) ? rawUrlLang[0] : rawUrlLang;
  const { lang, changeLang } = useLanguage();
  
  useEffect(() => {
    if (
      urlLang && 
      availableLanguages.includes(urlLang as AvailableLanguage) && 
      urlLang !== lang
    ) {
      changeLang(urlLang as AvailableLanguage);
    }
  }, [urlLang, lang, changeLang]);
  
  // This component doesn't render anything visible
  return null;
}
```

### 5. Combined Middleware (Clerk + i18n)

```typescript
// middleware.ts
import { clerkMiddleware, createRouteMatcher } from "@clerk/nextjs/server";
import { NextResponse } from 'next/server';
import { NextFetchEvent, NextRequest } from 'next/server';
import { AuthObject } from '@clerk/nextjs/server';

// Define available languages
type AvailableLanguage = 'en' | 'fr' | 'es';
const validLanguages: AvailableLanguage[] = ['en', 'fr', 'es'];
const defaultLanguage: AvailableLanguage = 'en';

// Public routes that don't require authentication
const isPublicRoute = createRouteMatcher(["/sign-in(.*)", "/sign-up(.*)", "/"]);

// Create a modified middleware that handles both auth and language
const combinedMiddleware = async (auth: AuthObject, request: NextRequest) => {
  const pathname: string = request.nextUrl.pathname;
  
  // Handle language for protected routes
  if (pathname.startsWith('/(protected)')) {
    // Check if language is already in URL
    const hasLang: boolean = validLanguages.some(
      (lang: AvailableLanguage) => pathname.match(new RegExp(`/(protected)/(?:${lang})(/|$)`)) !== null
    );
    
    // If no language in URL, redirect to add default language
    if (!hasLang) {
      // Get path after /(protected)/
      const pathAfterProtected: string = pathname.replace('/(protected)', '');
      
      // If path is just /(protected), redirect to /(protected)/en
      if (pathAfterProtected === '' || pathAfterProtected === '/') {
        return NextResponse.redirect(
          new URL(`/(protected)/${defaultLanguage}`, request.url)
        );
      }
      
      // Otherwise add language to the path
      return NextResponse.redirect(
        new URL(`/(protected)/${defaultLanguage}${pathAfterProtected}`, request.url)
      );
    }
  }
  
  // Handle Clerk authentication
  if (!isPublicRoute(request)) {
    await auth.protect();
  }
  
  return NextResponse.next();
};

// Apply the Clerk middleware with our combined function
export default clerkMiddleware(combinedMiddleware);

// Configure the middleware to run on specific paths
export const config = {
  matcher: [
    // Skip Next.js internals and all static files, unless found in search params
    "/((?!_next|[^?]*\\.(?:html?|css|js(?!on)|jpe?g|webp|png|gif|svg|ttf|woff2?|ico|csv|docx?|xlsx?|zip|webmanifest)).*)",
    // Always run for API routes
    "/(api|trpc)(.*)",
  ],
};
```

### 6. Protected Routes Layout

```tsx
// app/(protected)/[lang]/layout.tsx
// Server component - no 'use client' directive

import { SidebarProvider, SidebarTrigger } from "@/components/ui/sidebar";
import { UserButton } from "@clerk/nextjs";
import LanguageSynchronizer from "@/components/LanguageSynchronizer";
import React from "react";

interface SidebarLayoutProps {
  children: React.ReactNode;
}

const SidebarLayout = ({ children }: SidebarLayoutProps) => {
  return (
    <SidebarProvider>
      {/* Include the language synchronizer client component */}
      <LanguageSynchronizer />
      
      <main className="w-full">
        <div className="h-[5vh] flex items-center gap-2 border-sidebar-border bg-sidebar border p-2">
          <SidebarTrigger className="cursor-pointer" />
          <div className="ml-auto"></div>
          <UserButton />
        </div>

        {/* Main Content*/}
        <div className="border-sidebar-border border shadow overflow-y-scroll-auto min-h-[95vh] p-4">
          {children}
        </div>
      </main>
    </SidebarProvider>
  );
};

export default SidebarLayout;

export async function generateStaticParams() {
  return [
    { lang: "en" }, 
    { lang: "fr" }, 
    { lang: "es" }
  ];
}
```

### 7. Protected Route Default Redirect

```tsx
// app/(protected)/page.tsx
import { redirect } from 'next/navigation';

export default function ProtectedIndexPage() {
  redirect('/(protected)/en');
}
```

### 8. Example Protected Page with i18n

```tsx
// app/(protected)/[lang]/contract-checker/page.tsx
'use client';

import { useLanguage } from '@/context/LanguageContext';

export default function ContractChecker() {
  const { t } = useLanguage();
  
  return (
    <div>
      <h1>{t('protected.contractChecker.title')}</h1>
      <p>{t('protected.contractChecker.description')}</p>
      <button className="px-4 py-2 bg-blue-500 text-white rounded">
        {t('protected.contractChecker.uploadButton')}
      </button>
    </div>
  );
}
```

## Root Layout Setup

To make everything work together, your root layout should include both the Clerk provider and the Language provider:

```tsx
// app/layout.tsx
import { Inter } from "next/font/google";
import ConvexClerkProvider from "@/components/providers/ConvexClerkProvider";
import { LanguageProvider } from "@/context/LanguageContext";
import "./globals.css";

const inter = Inter({ subsets: ["latin"] });

export default function RootLayout({
  children,
}: {
  children: React.ReactNode;
}) {
  return (
    <html lang="en">
      <body className={inter.className}>
        <ConvexClerkProvider>
          <LanguageProvider>
            {children}
          </LanguageProvider>
        </ConvexClerkProvider>
      </body>
    </html>
  );
}
```

## Usage Guide

### 1. Accessing Translations in Components

Use the `useLanguage` hook in any client component:

```tsx
'use client';

import { useLanguage } from '@/context/LanguageContext';

export default function MyComponent() {
  const { t, lang, changeLang } = useLanguage();
  
  return (
    <div>
      <h1>{t('protected.some.translation.key')}</h1>
      <p>Current language: {lang}</p>
      <button onClick={() => changeLang('fr')}>Switch to French</button>
    </div>
  );
}
```

### 2. URL-based Navigation

When linking between protected pages, include the language parameter:

```tsx
import Link from 'next/link';
import { useParams } from 'next/navigation';

export function NavigationLinks() {
  const params = useParams();
  const lang = params.lang || 'en';
  
  return (
    <nav>
      <Link href={`/(protected)/${lang}/contract-checker`}>Contract Checker</Link>
      <Link href={`/(protected)/${lang}/legal-ai`}>Legal AI</Link>
    </nav>
  );
}
```

### 3. Adding New Languages

To add a new language:

1. Create a new translation file in `consts/translations/`
2. Add the language code to the `AvailableLanguage` type and `validLanguages` array in multiple files:
    - `context/LanguageContext.tsx`
    - `middleware.ts`
    - `components/LanguageSwitcher.tsx`
    - `components/LanguageSynchronizer.tsx`
3. Add the language to `generateStaticParams()` in `app/(protected)/[lang]/layout.tsx`

## Benefits & Considerations

### Benefits

- **SEO-friendly**: Language-specific URLs for content indexing
- **User-friendly**: Persistent language selection
- **Developer-friendly**: Centralized translation management
- **Minimal interference**: Works alongside Clerk authentication
- **Scalable**: Easy to add more languages and translation keys

### Considerations

- Only protected routes have URL-based language parameters
- Main page uses context-based translations without URL changes
- Language selection persists via localStorage
- Authentication flow remains untouched

## Troubleshooting

### Common Issues

1. **"Invalid page configuration"**: Don't use both "use client" and export `generateStaticParams()` in the same file. Use the LanguageSynchronizer pattern instead.
    
2. **TypeScript errors with useParams**: The `useParams` hook returns parameters that could be strings or arrays. Always handle both cases as shown in the LanguageSynchronizer component.
    
3. **Middleware not running**: Check the matcher configuration in middleware.ts. Ensure the matcher includes your protected routes pattern.
    
4. **Missing translations**: If a translation key doesn't exist, the `t` function returns the key itself. Add proper error handling and fallbacks in your components.