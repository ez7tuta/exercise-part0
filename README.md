# Notes App Diagrams

## 0.4: New note diagram

```mermaid
sequenceDiagram
    participant browser
    participant server
    Note right of browser: User writes note and clicks Save
    browser->>server: POST https://studies.cs.helsinki.fi/exampleapp/new_note (note content)
    activate server
    server-->>browser: Redirect / Reload page
    deactivate server
    browser->>server: GET https://studies.cs.helsinki.fi/exampleapp/notes
    activate server
    server-->>browser: HTML document
    deactivate server
    browser->>server: GET .../main.css
    server-->>browser: CSS file
    browser->>server: GET .../main.js
    server-->>browser: JS file
    browser->>server: GET .../data.json
    server-->>browser: Notes data (JSON)
    Note right of browser: Browser renders notes
```

## 0.5: Single page app diagram

```mermaid
sequenceDiagram
    participant browser
    participant server
    browser->>server: GET https://studies.cs.helsinki.fi/exampleapp/spa
    server-->>browser: HTML document
    browser->>server: GET .../main.css
    server-->>browser: CSS file
    browser->>server: GET .../spa.js
    server-->>browser: JS file
    browser->>server: GET .../data.json
    server-->>browser: Notes data (JSON)
    Note right of browser: Browser renders notes dynamically (no reload)
```

## 0.6: New note in Single page app diagram

```mermaid
sequenceDiagram
    participant browser
    participant server
    Note right of browser: User writes note and clicks Save
    browser->>server: POST https://studies.cs.helsinki.fi/exampleapp/new_note_spa (note as JSON)
    server-->>browser: Confirmation/OK
    Note right of browser: Browser updates notes list dynamically (no reload)
```
