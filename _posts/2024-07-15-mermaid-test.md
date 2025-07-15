---
layout: mermaid-post
title: Testing Mermaid Diagrams
date: 2024-07-15
toc: true
---

## Testing Mermaid Diagrams

This post tests the rendering of Mermaid diagrams on GitHub Pages.

### Flowchart Example

```mermaid
graph TD
    A[Start] --> B{Is it working?}
    B -->|Yes| C[Great!]
    B -->|No| D[Debug]
    D --> B
```

### Sequence Diagram Example

```mermaid
sequenceDiagram
    participant User
    participant System
    User->>System: Request data
    System->>User: Return data
    User->>System: Process data
    System->>User: Confirm processing
```

### Class Diagram Example

```mermaid
classDiagram
    class Animal {
        +name: string
        +age: int
        +makeSound(): void
    }
    class Dog {
        +breed: string
        +bark(): void
    }
    class Cat {
        +color: string
        +meow(): void
    }
    Animal <|-- Dog
    Animal <|-- Cat
```

### Gantt Chart Example

```mermaid
gantt
    title Project Schedule
    dateFormat  YYYY-MM-DD
    section Planning
    Requirements gathering :a1, 2024-07-01, 7d
    Design                 :a2, after a1, 10d
    section Development
    Implementation         :a3, after a2, 15d
    Testing                :a4, after a3, 7d
    section Deployment
    Deployment             :a5, after a4, 3d
```

## Conclusion

If you can see the diagrams above, then Mermaid is working correctly on your GitHub Pages site!
