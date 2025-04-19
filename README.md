# BOWTIE 🎀 Brighten Our World Through Intelligent Engineering

## Requirements Document for JavaScript/Node.js Application

This document outlines the detailed requirements for a JavaScript/Node.js application, structured into three layers: Server, Application, and User Interface. The application uses a simple in-memory key-value store with periodic synchronization, avoiding PouchDB/CouchDB. The design prioritizes flat objects, avoids inheritance, and uses descriptive naming for clarity in AI interpretation. The application is conceptualized as a story of objects working together, with minimal server dependency for scalability.

## Server Layer Requirements

The Server layer handles data storage, synchronization, and minimal processing to reduce costs and ensure scalability. It uses an in-memory key-value store with periodic disk persistence and synchronization across server instances.

### KeyValueStore

**Purpose**: Stores data as key-value pairs with path-based keys (e.g., `/users/alice/projects`) and JSON-serializable values.

**Functionality**:

- **SetKeyValue**: Stores a key-value pair with a revision number and UUID.
  - **Input**: Path (string), Value (JSON-serializable), Revision (integer), UUID (string).
  - **Output**: Confirmation or error if revision is outdated.
- **GetKeyValue**: Retrieves the latest value for a given path.
  - **Input**: Path (string).
  - **Output**: Latest value, revision, and UUID, or null if not found.
- **ListKeys**: Retrieves all keys under a path prefix.
  - **Input**: Path prefix (string).
  - **Output**: Array of matching keys.

**Revision Management**:

- Each value has a revision number (starting at 1) and UUID.
- Updates increment the revision unless a conflict occurs.
- Conflicts (same revision) are resolved by sorting UUIDs lexicographically; the highest UUID wins.
- Conflicts are logged with a conflict flag.

**Tree Structure**:

- Maintains an in-memory tree for efficient iteration (e.g., `/users/alice/projects` creates nodes for `users`, `alice`, `projects`).
- Supports iteration over keys under a path.

**Implementation**:

- In-memory storage using a Map.
- Periodic persistence to disk as JSON files every 5 minutes or on significant updates (e.g., 100 new keys).
- Uses Node.js `fs` module.

```javascript
// server/KeyValueStore.js
/**
 * Manages a key-value store with path-based keys and revision tracking.
 */
export class KeyValueStore {
  #store = new Map();
  #tree = new Map();
  #conflictLog = [];

  /**
   * Stores a key-value pair with revision and UUID.
   * @param {string} path - The key path (e.g., /users/alice/projects).
   * @param {any} value - JSON-serializable value.
   * @param {number} revision - Revision number.
   * @param {string} uuid - Unique identifier.
   * @returns {{ success: boolean, error?: string }} Confirmation or error.
   */
  setKeyValue(path, value, revision, uuid) {
    const existing = this.#store.get(path);
    if (existing && existing.revision >= revision) {
      if (existing.revision === revision && existing.uuid < uuid) {
        this.#store.set(path, { value, revision, uuid });
        this.#updateTree(path);
        return { success: true };
      }
      this.#conflictLog.push({ path, revision, uuid, timestamp: Date.now() });
      return { success: false, error: 'Revision conflict' };
    }
    this.#store.set(path, { value, revision, uuid });
    this.#updateTree(path);
    return { success: true };
  }

  /**
   * Retrieves the latest value for a path.
   * @param {string} path - The key path.
   * @returns {{ value: any, revision: number, uuid: string } | null} Value, revision, UUID, or null.
   */
  getKeyValue(path) {
    return this.#store.get(path) || null;
  }

  /**
   * Lists all keys under a path prefix.
   * @param {string} prefix - Path prefix (e.g., /users).
   * @returns {string[]} Array of matching keys.
   */
  listKeys(prefix) {
    const keys = [];
    for (const [path] of this.#store) {
      if (path.startsWith(prefix)) {
        keys.push(path);
      }
    }
    return keys;
  }

  /**
   * Updates the in-memory tree for a path.
   * @private
   * @param {string} path - The key path.
   */
  #updateTree(path) {
    const parts = path.split('/').filter(Boolean);
    let current = this.#tree;
    for (const part of parts) {
      if (!current.has(part)) {
        current.set(part, new Map());
      }
      current = current.get(part);
    }
  }
}
```

### SyncManager

**Purpose**: Synchronizes key-value data between servers and clients.

**Functionality**:

- **PushSync**: Sends updated key-value pairs.
  - **Input**: Array of `{Path, Value, Revision, UUID}`.
  - **Output**: Confirmation or error.
- **PullSync**: Retrieves updates.
  - **Input**: Last known revision per path.
  - **Output**: Array of updated `{Path, Value, Revision, UUID}`.
- **ConflictResolution**: Resolves conflicts using revision and UUID sorting.

**Implementation**:

- Uses WebSocket or HTTP.
- Synchronizes every 30 seconds or on significant updates.
- Ensures at-least-once delivery via a SyncLog.

```javascript
// server/SyncManager.js
/**
 * Manages synchronization of key-value data across servers and clients.
 */
export class SyncManager {
  #syncLog = [];
  #keyValueStore;

  /**
   * @param {KeyValueStore} keyValueStore - The key-value store to sync.
   */
  constructor(keyValueStore) {
    this.#keyValueStore = keyValueStore;
  }

  /**
   * Sends updated key-value pairs to other servers or clients.
   * @param {{ path: string, value: any, revision: number, uuid: string }[]} updates - Updates to sync.
   * @returns {{ success: boolean, error?: string }} Confirmation or error.
   */
  async pushSync(updates) {
    try {
      // Simulate WebSocket/HTTP sync
      for (const { path, value, revision, uuid } of updates) {
        const result = this.#keyValueStore.setKeyValue(path, value, revision, uuid);
        if (!result.success) {
          this.#syncLog.push({ path, value, revision, uuid, timestamp: Date.now() });
          return { success: false, error: result.error };
        }
      }
      return { success: true };
    } catch (error) {
      this.#syncLog.push({ updates, timestamp: Date.now(), error: error.message });
      return { success: false, error: error.message };
    }
  }

  /**
   * Retrieves updates from other servers or clients.
   * @param {{ path: string, revision: number }[]} lastRevisions - Last known revisions.
   * @returns {{ path: string, value: any, revision: number, uuid: string }[]} Updated data.
   */
  async pullSync(lastRevisions) {
    const updates = [];
    for (const { path, revision } of lastRevisions) {
      const data = this.#keyValueStore.getKeyValue(path);
      if (data && data.revision > revision) {
        updates.push({ path, ...data });
      }
    }
    return updates;
  }
}
```

### ServerController

**Purpose**: Manages server operations and exposes APIs.

**Functionality**:

- **StartServer**: Initializes server and loads data.
- **HandleClientRequest**: Processes client requests.
  - **Endpoints**:
    - `POST /set`: Sets a key-value pair.
    - `GET /get`: Retrieves a value.
    - `GET /list`: Lists keys.
    - `POST /sync`: Handles sync requests.
- **MonitorHealth**: Tracks uptime and logs errors.

**Implementation**:

- Uses Express on port 3000.
- Logs to `server.log` with Winston.

```javascript
// server/ServerController.js
import express from 'express';
import winston from 'winston';

/**
 * Manages server operations and client APIs.
 */
export class ServerController {
  #app = express();
  #keyValueStore;
  #syncManager;
  #logger;

  /**
   * @param {KeyValueStore} keyValueStore - The key-value store.
   * @param {SyncManager} syncManager - The sync manager.
   */
  constructor(keyValueStore, syncManager) {
    this.#keyValueStore = keyValueStore;
    this.#syncManager = syncManager;
    this.#logger = winston.createLogger({
      transports: [new winston.transports.File({ filename: 'server.log' })],
    });
    this.#setupRoutes();
  }

  /**
   * Sets up Express routes.
   * @private
   */
  #setupRoutes() {
    this.#app.use(express.json());
    this.#app.post('/set', async (req, res) => {
      const { path, value, revision, uuid } = req.body;
      const result = this.#keyValueStore.setKeyValue(path, value, revision, uuid);
      res.json(result);
    });
    this.#app.get('/get', async (req, res) => {
      const { path } = req.query;
      const data = this.#keyValueStore.getKeyValue(path);
      res.json(data);
    });
    this.#app.get('/list', async (req, res) => {
      const { prefix } = req.query;
      const keys = this.#keyValueStore.listKeys(prefix);
      res.json(keys);
    });
    this.#app.post('/sync', async (req, res) => {
      const { updates } = req.body;
      const result = await this.#syncManager.pushSync(updates);
      res.json(result);
    });
  }

  /**
   * Starts the server.
   * @param {number} [port=3000] - Port to listen on.
   */
  startServer(port = 3000) {
    this.#app.listen(port, () => {
      this.#logger.info(`Server running on port ${port}`);
    });
  }

  /**
   * Logs server health and errors.
   */
  monitorHealth() {
    // Simulated health check
    this.#logger.info(`Server uptime: ${process.uptime()}s`);
  }
}
```

## Application Layer Requirements

The Application layer manages core logic, state, and operations with flat objects, designed to be AI-friendly and independent of the UI.

### Application

**Purpose**: Coordinates application components.

**Properties**:

- **Toolboxes**: Reference to Toolboxes.
- **Scenery**: Reference to Scenery.
- **Commander**: Reference to Commander.
- **State**: In-memory key-value store.

**Functionality**:

- **Initialize**: Sets up components and syncs state.
- **SyncWithServer**: Synchronizes State with Server.
- **ExecuteCommand**: Routes commands to Commander.

**Implementation**:

- Uses a Map for State.
- Communicates via HTTP/WebSocket.

```javascript
// application/Application.js
/**
 * Central hub for application components and state.
 */
export class Application {
  #toolboxes;
  #scenery;
  #commander;
  #state = new Map();

  /**
   * @param {Toolboxes} toolboxes - Toolboxes instance.
   * @param {Scenery} scenery - Scenery instance.
   * @param {Commander} commander - Commander instance.
   */
  constructor(toolboxes, scenery, commander) {
    this.#toolboxes = toolboxes;
    this.#scenery = scenery;
    this.#commander = commander;
  }

  /**
   * Initializes components and syncs state.
   * @returns {Promise<void>}
   */
  async initialize() {
    // Simulate server sync
    await this.syncWithServer();
  }

  /**
   * Synchronizes state with server.
   * @returns {Promise<void>}
   */
  async syncWithServer() {
    // Simulate HTTP/WebSocket sync
    const updates = await fetch('/sync', { method: 'POST', body: JSON.stringify([]) }).then(res => res.json());
    for (const { path, value } of updates) {
      this.#state.set(path, value);
    }
  }

  /**
   * Executes a command via Commander.
   * @param {string} commandName - Command name.
   * @param {object} args - Command arguments.
   * @returns {{ result: any, error?: string }} Result or error.
   */
  executeCommand(commandName, args) {
    return this.#commander.executeCommand(commandName, args);
  }
}
```

### Toolboxes

**Purpose**: Manages collections of Tools.

**Properties**:

- **Tools**: Map of Tool names to Tool objects.

**Functionality**:

- **AddTool**: Registers a Tool.
- **GetTool**: Retrieves a Tool.
- **ExecuteToolAction**: Runs a Tool action.

```javascript
// application/Toolboxes.js
/**
 * Manages collections of Tools.
 */
export class Toolboxes {
  #tools = new Map();

  /**
   * Registers a new Tool.
   * @param {string} toolName - Tool name.
   * @param {object} tool - Tool object.
   */
  addTool(toolName, tool) {
    this.#tools.set(toolName, tool);
  }

  /**
   * Retrieves a Tool by name.
   * @param {string} toolName - Tool name.
   * @returns {object | null} Tool object or null.
   */
  getTool(toolName) {
    return this.#tools.get(toolName) || null;
  }

  /**
   * Runs a Tool action.
   * @param {string} toolName - Tool name.
   * @param {string} actionName - Action name.
   * @param {object} args - Action arguments.
   * @returns {{ result: any, error?: string }} Result or error.
   */
  executeToolAction(toolName, actionName, args) {
    const tool = this.getTool(toolName);
    if (!tool || !tool.actions[actionName]) {
      return { error: 'Tool or action not found' };
    }
    return tool.executeAction(actionName, args);
  }
}
```

### Tools

**Purpose**: Handles specific tasks (e.g., project management).

**Properties**:

- **Name**: Unique identifier.
- **Actions**: Map of action names to functions.

**Functionality**:

- **ExecuteAction**: Runs an action.

```javascript
// application/Tools.js
/**
 * Represents a functional unit with specific actions.
 */
export class Tool {
  #name;
  #actions;

  /**
   * @param {string} name - Tool name.
   * @param {object} actions - Map of action names to functions.
   */
  constructor(name, actions) {
    this.#name = name;
    this.#actions = actions;
  }

  /**
   * Runs a specific action.
   * @param {string} actionName - Action name.
   * @param {object} args - Action arguments.
   * @returns {{ result: any, error?: string }} Result or error.
   */
  executeAction(actionName, args) {
    const action = this.#actions[actionName];
    if (!action) {
      return { error: 'Action not found' };
    }
    return { result: action(args) };
  }
}
```

### Scenery

**Purpose**: Manages application context via Scenes.

**Properties**:

- **Scenes**: Map of Scene names to Scene objects.
- **ActiveScene**: Current Scene.

**Functionality**:

- **AddScene**: Registers a Scene.
- **SetActiveScene**: Sets the active Scene.
- **GetSceneData**: Retrieves Scene data.

```javascript
// application/Scenery.js
/**
 * Manages application context through Scenes.
 */
export class Scenery {
  #scenes = new Map();
  #activeScene = null;

  /**
   * Registers a new Scene.
   * @param {string} sceneName - Scene name.
   * @param {object} scene - Scene object.
   */
  addScene(sceneName, scene) {
    this.#scenes.set(sceneName, scene);
  }

  /**
   * Sets the active Scene.
   * @param {string} sceneName - Scene name.
   */
  setActiveScene(sceneName) {
    if (this.#scenes.has(sceneName)) {
      this.#activeScene = this.#scenes.get(sceneName);
    }
  }

  /**
   * Retrieves data for a Scene.
   * @param {string} sceneName - Scene name.
   * @returns {object | null} Scene data or null.
   */
  getSceneData(sceneName) {
    const scene = this.#scenes.get(sceneName);
    return scene ? scene.getData() : null;
  }
}
```

### Scenes

**Purpose**: Represents a specific context (e.g., project view).

**Properties**:

- **Name**: Unique identifier.
- **Data**: JSON-serializable state.

**Functionality**:

- **UpdateData**: Updates Scene data.
- **GetData**: Returns Scene data.

```javascript
// application/Scenes.js
/**
 * Represents a specific application context.
 */
export class Scene {
  #name;
  #data;

  /**
   * @param {string} name - Scene name.
   * @param {object} data - Initial Scene data.
   */
  constructor(name, data) {
    this.#name = name;
    this.#data = data;
  }

  /**
   * Updates Scene data.
   * @param {object} data - New data.
   */
  updateData(data) {
    this.#data = { ...this.#data, ...data };
  }

  /**
   * Returns Scene data.
   * @returns {object} Current data.
   */
  getData() {
    return this.#data;
  }
}
```

### Commander

**Purpose**: Manages command execution and history.

**Properties**:

- **Commands**: Map of Command names to Command objects.
- **History**: Reference to History.

**Functionality**:

- **ExecuteCommand**: Runs a command and adds to History.
- **Undo**: Undoes the latest command.
- **Redo**: Redoes the next command.

```javascript
// application/Commander.js
/**
 * Manages command execution and history.
 */
export class Commander {
  #commands = new Map();
  #history;

  /**
   * @param {History} history - History instance.
   */
  constructor(history) {
    this.#history = history;
  }

  /**
   * Registers a command.
   * @param {string} commandName - Command name.
   * @param {object} command - Command object.
   */
  addCommand(commandName, command) {
    this.#commands.set(commandName, command);
  }

  /**
   * Executes a command and adds to history.
   * @param {string} commandName - Command name.
   * @param {object} args - Command arguments.
   * @returns {{ result: any, error?: string }} Result or error.
   */
  executeCommand(commandName, args) {
    const command = this.#commands.get(commandName);
    if (!command) {
      return { error: 'Command not found' };
    }
    const result = command.run(args);
    this.#history.addEvent(command, command, args);
    return result;
  }

  /**
   * Undoes the latest command.
   * @returns {{ result: any, error?: string }} Result or error.
   */
  undo() {
    return this.#history.undo();
  }

  /**
   * Redoes the next command.
   * @returns {{ result: any, error?: string }} Result or error.
   */
  redo() {
    return this.#history.redo();
  }
}
```

### Commands

**Purpose**: Represents executable actions with undo/redo.

**Properties**:

- **Name**: Unique identifier.
- **Execute**: Function to perform the command.
- **Undo**: Function to reverse the command.

**Functionality**:

- **Run**: Executes the command.
- **RunUndo**: Executes the undo action.

```javascript
// application/Commands.js
/**
 * Represents an executable command with undo/redo.
 */
export class Command {
  #name;
  #execute;
  #undo;

  /**
   * @param {string} name - Command name.
   * @param {function} execute - Execute function.
   * @param {function} undo - Undo function.
   */
  constructor(name, execute, undo) {
    this.#name = name;
    this.#execute = execute;
    this.#undo = undo;
  }

  /**
   * Executes the command.
   * @param {object} args - Command arguments.
   * @returns {{ result: any, error?: string }} Result or error.
   */
  run(args) {
    return { result: this.#execute(args) };
  }

  /**
   * Executes the undo action.
   * @param {object} args - Command arguments.
   * @returns {{ result: any, error?: string }} Result or error.
   */
  runUndo(args) {
    return { result: this.#undo(args) };
  }
}
```

### History

**Purpose**: Tracks command execution for undo/redo.

**Properties**:

- **Events**: Array of Event objects.
- **CurrentIndex**: Current position in history.

**Functionality**:

- **AddEvent**: Adds a new event.
- **Undo**: Undoes the current event.
- **Redo**: Redoes the next event.

```javascript
// application/History.js
/**
 * Tracks command execution events for undo/redo.
 */
export class History {
  #events = [];
  #currentIndex = -1;

  /**
   * Adds a new event.
   * @param {Command} redoCommand - Redo command.
   * @param {Command} undoCommand - Undo command.
   * @param {object} args - Command arguments.
   */
  addEvent(redoCommand, undoCommand, args) {
    this.#events.splice(this.#currentIndex + 1);
    this.#events.push({ redoCommand, undoCommand, args, active: true });
    this.#currentIndex++;
  }

  /**
   * Undoes the current event.
   * @returns {{ result: any, error?: string }} Result or error.
   */
  undo() {
    if (this.#currentIndex < 0) {
      return { error: 'Nothing to undo' };
    }
    const event = this.#events[this.#currentIndex];
    event.active = false;
    this.#currentIndex--;
    return event.undoCommand.runUndo(event.args);
  }

  /**
   * Redoes the next event.
   * @returns {{ result: any, error?: string }} Result or error.
   */
  redo() {
    if (this.#currentIndex >= this.#events.length - 1) {
      return { error: 'Nothing to redo' };
    }
    this.#currentIndex++;
    const event = this.#events[this.#currentIndex];
    event.active = true;
    return event.redoCommand.run(event.args);
  }
}
```

## User Interface Layer Requirements

The UI layer visualizes the Application layer using WebComponents and Signals for reactivity, decoupled from the Application.

### UserInterface

**Purpose**: Coordinates WebComponents and user interactions.

**Properties**:

- **Components**: Map of component names to WebComponent instances.
- **Signals**: Map of signal names to Signal objects.

**Functionality**:

- **Initialize**: Sets up WebComponents and Signals.
- **Render**: Updates UI based on state.
- **HandleUserInput**: Translates user actions to commands.

```javascript
// ui/UserInterface.js
/**
 * Coordinates WebComponents and user interactions.
 */
export class UserInterface {
  #components = new Map();
  #signals = new Map();

  /**
   * Initializes WebComponents and Signals.
   */
  initialize() {
    // Register custom elements
    customElements.define('application-component', ApplicationComponent);
    // Add components and signals
    this.#components.set('app', new ApplicationComponent());
    this.#signals.set('activeScene', new Signal(null));
  }

  /**
   * Updates UI based on state changes.
   */
  render() {
    for (const component of this.#components.values()) {
      component.render?.();
    }
  }

  /**
   * Translates user actions to Application commands.
   * @param {string} componentName - Component name.
   * @param {string} action - Action name.
   * @param {object} data - Action data.
   */
  handleUserInput(componentName, action, data) {
    const component = this.#components.get(componentName);
    if (component?.handleEvents) {
      component.handleEvents(action, data);
    }
  }
}
```

### Signals

**Purpose**: Manages reactive state.

**Properties**:

- **Value**: Current value.
- **Subscribers**: Array of callbacks.

**Functionality**:

- **SetValue**: Updates value and notifies subscribers.
- **Subscribe**: Adds a callback.
- **Unsubscribe**: Removes a callback.

```javascript
// ui/Signals.js
/**
 * Manages reactive state with subscriptions.
 */
export class Signal {
  #value;
  #subscribers = [];

  /**
   * @param {any} initialValue - Initial signal value.
   */
  constructor(initialValue) {
    this.#value = initialValue;
  }

  /**
   * Updates the signal value and notifies subscribers.
   * @param {any} value - New value.
   */
  setValue(value) {
    this.#value = value;
    for (const callback of this.#subscribers) {
      callback(value);
    }
  }

  /**
   * Adds a callback for value changes.
   * @param {function} callback - Callback function.
   * @returns {function} Unsubscribe function.
   */
  subscribe(callback) {
    this.#subscribers.push(callback);
    return () => this.unsubscribe(callback);
  }

  /**
   * Removes a callback.
   * @param {function} callback - Callback to remove.
   */
  unsubscribe(callback) {
    this.#subscribers = this.#subscribers.filter(cb => cb !== callback);
  }
}
```

### ApplicationComponent

**Purpose**: Visualizes the entire Application state.

**Functionality**:

- **Render**: Displays Toolboxes and Scenery.
- **HandleEvents**: Listens for user interactions.

```javascript
// ui/ApplicationComponent.js
/**
 * WebComponent for the entire application.
 */
export class ApplicationComponent extends HTMLElement {
  #shadow;

  constructor() {
    super();
    this.#shadow = this.attachShadow({ mode: 'open' });
  }

  /**
   * Renders the component.
   */
  render() {
    this.#shadow.innerHTML = `
      <toolbox-component></toolbox-component>
      <scene-component></scene-component>
    `;
  }

  /**
   * Handles user events.
   * @param {string} action - Action name.
   * @param {object} data - Event data.
   */
  handleEvents(action, data) {
    // Route to Application
    console.log(`ApplicationComponent: ${action}`, data);
  }

  connectedCallback() {
    this.render();
  }
}
```

### ToolBoxComponent

**Purpose**: Displays a Toolbox and its Tools.

**Functionality**:

- **Render**: Lists Tools and actions.
- **HandleEvents**: Executes Tool actions.

```javascript
// ui/ToolBoxComponent.js
/**
 * WebComponent for a Toolbox.
 */
export class ToolBoxComponent extends HTMLElement {
  #shadow;

  constructor() {
    super();
    this.#shadow = this.attachShadow({ mode: 'open' });
  }

  /**
   * Renders the component.
   */
  render() {
    this.#shadow.innerHTML = `
      <div>Toolbox: <button data-action="execute">Run Tool</button></div>
    `;
  }

  /**
   * Handles user events.
   * @param {string} action - Action name.
   * @param {object} data - Event data.
   */
  handleEvents(action, data) {
    if (action === 'execute') {
      // Route to Toolboxes.executeToolAction
      console.log('ToolBoxComponent: execute', data);
    }
  }

  connectedCallback() {
    this.render();
    this.#shadow.querySelector('button').addEventListener('click', () =>
      this.handleEvents('execute', {})
    );
  }
}
```

### SceneComponent

**Purpose**: Visualizes a Scene’s data.

**Functionality**:

- **Render**: Displays Scene data.
- **HandleEvents**: Updates Scene data.

```javascript
// ui/SceneComponent.js
/**
 * WebComponent for a Scene.
 */
export class SceneComponent extends HTMLElement {
  #shadow;

  constructor() {
    super();
    this.#shadow = this.attachShadow({ mode: 'open' });
  }

  /**
   * Renders the component.
   */
  render() {
    this.#shadow.innerHTML = `<div>Scene: <input data-action="update" /></div>`;
  }

  /**
   * Handles user events.
   * @param {string} action - Action name.
   * @param {object} data - Event data.
   */
  handleEvents(action, data) {
    if (action === 'update') {
      // Route to Scenery.updateData
      console.log('SceneComponent: update', data);
    }
  }

  connectedCallback() {
    this.render();
    this.#shadow.querySelector('input').addEventListener('input', (e) =>
      this.handleEvents('update', { value: e.target.value })
    );
  }
}
```

### CommanderComponent

**Purpose**: Provides a command-line interface and undo/redo buttons.

**Functionality**:

- **Render**: Displays input and buttons.
- **HandleEvents**: Executes commands or undo/redo.

```javascript
// ui/CommanderComponent.js
/**
 * WebComponent for command execution.
 */
export class CommanderComponent extends HTMLElement {
  #shadow;

  constructor() {
    super();
    this.#shadow = this.attachShadow({ mode: 'open' });
  }

  /**
   * Renders the component.
   */
  render() {
    this.#shadow.innerHTML = `
      <div>
        <input data-action="command" placeholder="Enter command" />
        <button data-action="undo">Undo</button>
        <button data-action="redo">Redo</button>
      </div>
    `;
  }

  /**
   * Handles user events.
   * @param {string} action - Action name.
   * @param {object} data - Event data.
   */
  handleEvents(action, data) {
    if (action === 'command') {
      // Route to Commander.executeCommand
      console.log('CommanderComponent: command', data);
    } else if (action === 'undo') {
      // Route to Commander.undo
      console.log('CommanderComponent: undo');
    } else if (action === 'redo') {
      // Route to Commander.redo
      console.log('CommanderComponent: redo');
    }
  }

  connectedCallback() {
    this.render();
    this.#shadow.querySelector('[data-action="command"]').addEventListener('input', (e) =>
      this.handleEvents('command', { value: e.target.value })
    );
    this.#shadow.querySelector('[data-action="undo"]').addEventListener('click', () =>
      this.handleEvents('undo', {})
    );
    this.#shadow.querySelector('[data-action="redo"]').addEventListener('click', () =>
      this.handleEvents('redo', {})
    );
  }
}
```

## Information Flow

**User Interface to Application**:

- User interactions trigger commands via `UserInterface.handleUserInput`.
- Commands are sent to `Application.executeCommand`, routed to `Commander.executeCommand`.
- Commands update `Application.State` and may trigger `Scenery.setActiveScene` or `Toolboxes.executeToolAction`.

**Application to Server**:

- State changes in `Application.State` are synced with `Server.KeyValueStore` via `Application.syncWithServer`.
- `SyncManager` propagates data to other servers/clients.

**Server to Application**:

- `SyncManager.pullSync` retrieves updates.
- Application updates `State`, propagating to `Scenery` and `Toolboxes`.

**Application to User Interface**:

- State changes trigger `Signal` updates.
- WebComponents re-render based on `Signal` subscriptions.

## Non-Functional Requirements

- **Scalability**: Server minimizes processing for 10,000 concurrent users.
- **Reliability**: `SyncManager` ensures at-least-once delivery; conflicts logged.
- **Performance**: UI updates within 100ms; server syncs within 1s.
- **Maintainability**: Flat objects and descriptive names for AI compatibility.
- **Portability**: Runs in modern browsers and Node.js 18+.

## Implementation Notes

**Technologies**:

- JavaScript (ES Modules).
- Node.js with Express.
- WebComponents for UI.
- Custom `Signal` or @preact/signals.

**Directory Structure**:

```
/src
├── /server
│   ├── KeyValueStore.js
│   ├── SyncManager.js
│   └── ServerController.js
├── /application
│   ├── Application.js
│   ├── Toolboxes.js
│   ├── Tools.js
│   ├── Scenery.js
│   ├── Scenes.js
│   ├── Commander.js
│   ├── Commands.js
│   └── History.js
├── /ui
│   ├── UserInterface.js
│   ├── Signals.js
│   ├── ApplicationComponent.js
│   ├── ToolBoxComponent.js
│   ├── SceneComponent.js
│   └── CommanderComponent.js
└── /public
    └── index.html
```

### Linux/macOS

Open your terminal and use the following commands:

```bash
# Navigate to your desired parent directory where you wish to create the /src directory, then:
mkdir -p src/server src/application src/ui src/public

# Now create the files in their respective directories
touch src/server/KeyValueStore.js src/server/SyncManager.js src/server/ServerController.js
touch src/application/Application.js src/application/Toolboxes.js src/application/Tools.js
touch src/application/Scenery.js src/application/Scenes.js src/application/Commander.js
touch src/application/Commands.js src/application/History.js
touch src/ui/UserInterface.js src/ui/Signals.js src/ui/ApplicationComponent.js
touch src/ui/ToolBoxComponent.js src/ui/SceneComponent.js src/ui/CommanderComponent.js
touch src/public/index.html

```

**Dependencies**:

- `express` (`npm install express`)
- `winston` (`npm install winston`)

This document provides an AI-friendly blueprint with detailed class implementations, ensuring clarity and precision for development.
