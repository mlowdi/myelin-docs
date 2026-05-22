# Admin Command Channel

The Admin Command Channel provides a secure, role-based infrastructure for executing system-level commands within the Myelin runtime.

## Purpose and Motivation

As Myelin operates autonomously, operators need a way to inspect system state, diagnose issues, and manage resources (like backends or budget) without direct SSH access or manual database manipulation. The Command Channel centralizes this control while ensuring that only authorized principals can execute sensitive operations.

## Module Structure

The Admin subsystem is composed of three primary modules:

*   **`Myelin.Admin`**: (Namespace) Entry point for administrative logic.
*   **`Myelin.Admin.Principals`**: An ETS-backed store for administrative identities. It maps identifiers (e.g., Telegram user IDs) to roles (`:admin`, `:operator`).
*   **`Myelin.Admin.CommandChannel`**: The dispatcher that receives, authorizes, and executes commands. It bridges the administrative request with the internal system modules.

## Data Flow

1.  **Ingress**: An interface (like Telegram) receives a message starting with `/`.
2.  **Dispatch**: The interface calls `Myelin.Admin.CommandChannel.dispatch/4` with the command, arguments, and the sender's identifier.
3.  **Authorization**: `CommandChannel` queries `Admin.Principals` to verify the sender's role.
4.  **Execution**: If authorized (currently `:admin` only), the command is handled by internal logic which may query the `StateMachine`, `BackendPool`, or `Budget`.
5.  **Response**: The result text is returned to the interface for delivery to the user.

## Role Model

Currently, the system supports a simple role model:
*   **`:admin`**: Full access to all administrative commands.
*   **`:operator`**: (Reserved) For future restricted access levels.

Only principals explicitly registered in `Admin.Principals` with the `:admin` role can execute commands.

## MVP Command Set

*   `/status`: Returns the current attention state and active session details.
*   `/backends`: Lists all registered backends, their status, load, and latency.
*   `/budget`: Displays the 24h spending, remaining budget, and conservation mode status.
*   `/kick <name>`: Forces a specific backend to `:online` status.

## OutputEvent Integration

Commands processed through the CommandChannel often result in responses delivered via the same interface. These responses are flagged with:
*   **Source**: `:admin`
*   **Priority Bypass**: Admin responses should typically bypass standard salience or rate-limiting filters to ensure the operator receives immediate feedback.

## Registration Flow Overview

Principals must be registered before they can use the command channel. Currently, this is done via:
1.  Manual registration during bootstrap or via the Elixir console.
2.  Future Wave 2 features may include an invitation or "first-admin" setup flow.

## Wave 2 Status: Telegram Wiring

In Wave 2, the Telegram interface is being updated to recognize admin commands. When a message from a known admin principal is received, it is routed to the `CommandChannel` instead of the standard `EventPipeline`.
