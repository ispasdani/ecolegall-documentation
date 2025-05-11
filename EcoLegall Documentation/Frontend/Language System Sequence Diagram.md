sequenceDiagram
    actor User
    participant LS as LanguageSwitcher
    participant LC as LanguageContext
    participant URL as Browser URL
    participant LSY as LanguageSynchronizer
    participant Components as App Components
    participant Storage as LocalStorage
    
    Note over User,Storage: Language Change Flow
    User->>LS: Select language (e.g., Romanian)
    LS->>LC: changeLang("ro")
    LC->>LC: setLang("ro")
    LC->>Storage: Save preference
    LS->>URL: router.push("/ro/current-path")
    
    Note over User,Storage: Page Navigation Flow
    User->>URL: Navigate to new page
    URL->>LSY: URL changes (/ro/new-path)
    LSY->>LC: Check if context matches URL
    LSY->>LC: changeLang("ro") if needed
    
    Note over User,Storage: Component Translation Flow
    Components->>LC: useLanguage()
    LC->>Components: Provide { lang, t }
    Components->>Components: Render with t("key")
    
    Note over User,Storage: Initial Page Load
    URL->>LSY: Parse URL language
    Storage->>LC: Load saved preference
    LSY->>LC: Sync URL and context languages
    LC->>Components: Provide translations