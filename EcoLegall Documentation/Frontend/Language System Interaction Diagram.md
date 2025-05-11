graph TD
    A[User Interaction] -->|Selects Language| B[LanguageSwitcher]
    B -->|1. changeLang| C[LanguageContext]
    B -->|2. router.push| D[URL/Browser History]
    
    E[Page Load/Navigation] -->|Changes URL| F[Next.js Router]
    F -->|Provides params| G[LanguageSynchronizer]
    G -->|Reads lang param| D
    G -->|If URL lang ≠ Context lang| C
    
    H[Components] -->|useLanguage()| C
    C -->|Provides lang & t()| H
    
    I[LocalStorage] -->|Initial load| C
    C -->|Persists preference| I
    
    subgraph "State Management"
    C
    I
    end
    
    subgraph "URL Management"
    D
    F
    end
    
    subgraph "User Interface"
    A
    B
    H
    end
    
    style C fill:#f9f,stroke:#333,stroke-width:2px
    style D fill:#bbf,stroke:#333,stroke-width:2px
    style B fill:#bfb,stroke:#333,stroke-width:2px
    style G fill:#fbf,stroke:#333,stroke-width:2px