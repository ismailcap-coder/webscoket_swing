# UML Diagrams — Allegro WebSocket Swing PoC

## Class Diagram

```mermaid
classDiagram
    class Main_com {
        +main(String[] args)$
    }
    class Main_websocket {
        -CountDownLatch latch$
        -JFrame frame$
        -JTextField tf_name, tf_first, tf_dob...$
        -JRadioButton rb_female, rb_male, rb_diverse$
        -JsonParserFactory jsonParserFactory$
        +main(String[] args)$
        -initUI()$
        +toSearchResult(String json)$ SearchResult
    }
    class WebsocketClientEndpoint {
        -Session userSession
        +WebsocketClientEndpoint(URI)
        +onOpen(Session)
        +onClose(Session, CloseReason)
        +onMessage(String json)
        +sendMessage(String)
        +extract(String json)$ Message
    }
    class Message {
        +String target
        +String content
    }
    class SearchResult {
        +String name
        +String first
        +String dob
        +String zip
        +String ort
        +String street
        +String hausnr
        +String ze_iban
        +String ze_bic
        +String ze_valid_from
    }
    class PocView {
        +JFrame frame
        +JTextArea textArea
        +JTextField name, firstName, dateOfBirth
        +JTextField zip, ort, street, iban, bic, validFrom
        +JRadioButton female, male, diverse
        +ButtonGroup gender
        +JButton button
        +PocView()
        -initUI()
    }
    class PocPresenter {
        -PocView view
        -PocModel model
        +PocPresenter(PocView, PocModel, EventEmitter)
        -bind(JTextComponent, ModelProperties)
        -bind(JRadioButton, ModelProperties)
        -initializeBindings()
    }
    class PocModel {
        +Map~ModelProperties,ValueModel~ model
        -HttpBinService httpBinService
        -EventEmitter eventEmitter
        +PocModel(EventEmitter)
        +action() throws IOException, InterruptedException
    }
    class HttpBinService {
        +String URL = "http://localhost:8080"$
        +String PATH = "/post"$
        +post(Map~String,String~) String
    }
    class EventEmitter {
        -List~EventListener~ listeners
        +subscribe(EventListener)
        +emit(String eventData)
    }
    class EventListener {
        <<interface>>
        +onEvent(String eventData)
    }
    class ModelProperties {
        <<enumeration>>
        TEXT_AREA
        FIRST_NAME
        LAST_NAME
        DATE_OF_BIRTH
        ZIP
        ORT
        STREET
        IBAN
        BIC
        VALID_FROM
        FEMALE
        MALE
        DIVERSE
    }
    class ValueModel~T~ {
        -T field
        +ValueModel(T)
        +getField() T
        +setField(T)
    }

    Main_com --> PocView : creates
    Main_com --> EventEmitter : creates
    Main_com --> PocModel : creates
    Main_com --> PocPresenter : creates
    Main_websocket +-- WebsocketClientEndpoint
    Main_websocket +-- Message
    Main_websocket +-- SearchResult
    PocPresenter --> PocView : updates
    PocPresenter --> PocModel : calls action()
    PocPresenter --> EventEmitter : subscribes
    PocModel --> HttpBinService : uses
    PocModel --> EventEmitter : emits
    PocModel --> ValueModel : stores in map
    PocModel --> ModelProperties : keys
    EventEmitter --> EventListener : notifies
```

---

## Sequence Diagram — Customer Transfer Flow

```mermaid
sequenceDiagram
    actor User
    participant VueSearch as Vue Search.vue
    participant WSServer as Node.js WS Server :1337
    participant SwingClient as Swing websocket/Main.java
    participant SwingUI as Swing JFrame (Allegro)

    User->>VueSearch: Enter search criteria (name/zip/etc.)
    VueSearch->>VueSearch: searchPerson() — filter search_space[]
    VueSearch-->>User: Display results table

    User->>VueSearch: Select customer row
    VueSearch->>VueSearch: selectResult(item)

    User->>VueSearch: Select Zahlungsempfänger row
    VueSearch->>VueSearch: zahlungsempfaengerSelected(item)

    User->>VueSearch: Click "Nach ALLEGRO übernehmen"
    VueSearch->>VueSearch: sendMessage(selected_result, 'textfield')
    VueSearch->>WSServer: send JSON {target:"textfield", content:{customer+zahlungsempfaenger}}

    WSServer->>SwingClient: broadcast to all clients (verbatim)
    SwingClient->>SwingClient: onMessage(json) → extract() → Message{target,content}
    SwingClient->>SwingClient: toSearchResult(json) → SearchResult
    SwingClient->>SwingUI: setText() on all 10 fields
    SwingUI-->>User: Form populated with customer data
```

---

## Sequence Diagram — Allegro Form Submission (MVP Path)

```mermaid
sequenceDiagram
    actor User
    participant PocView as PocView (Swing)
    participant PocPresenter as PocPresenter
    participant PocModel as PocModel
    participant HttpBin as httpbin :8080/post
    participant EventEmitter as EventEmitter

    User->>PocView: Type in field (keystroke)
    PocView->>PocPresenter: DocumentListener.insertUpdate()
    PocPresenter->>PocModel: model.get(prop).setField(content)

    User->>PocView: Click "Anordnen"
    PocView->>PocPresenter: ActionListener callback
    PocPresenter->>PocModel: action()
    PocModel->>PocModel: collect all 13 fields from EnumMap
    PocModel->>HttpBin: POST /post {JSON of all fields}
    HttpBin-->>PocModel: 200 OK + response body
    PocModel->>EventEmitter: emit(responseBody)
    EventEmitter->>PocPresenter: onEvent(eventData)
    PocPresenter->>PocView: textArea.setText(eventData)
    PocPresenter->>PocView: clear all fields, reset gender=female
```

---

## Use Case Diagram

```mermaid
graph LR
    User((User / Sachbearbeiter))
    AllegroOp((Allegro System))

    subgraph Vue Search Frontend
        UC1[Search Customer by Name]
        UC2[Search Customer by ZIP/City/Street]
        UC3[Select Customer from Results]
        UC4[Select Zahlungsempfänger]
        UC5[Transfer to Allegro]
        UC6[Send Textarea Message]
    end

    subgraph Java Swing Allegro
        UC7[Receive Customer Data via WebSocket]
        UC8[View Pre-filled Form Fields]
        UC9[Edit Form Fields]
        UC10[Submit Form via Anordnen]
        UC11[View API Response]
    end

    User --> UC1
    User --> UC2
    User --> UC3
    User --> UC4
    User --> UC5
    User --> UC6
    User --> UC9
    User --> UC10

    UC5 -->|WebSocket message| UC7
    UC7 --> UC8
    UC10 -->|HTTP POST| AllegroOp
    AllegroOp -->|Response| UC11
```
