# ADK x402 Payment Protocol Demo

This project demonstrates a complete, end-to-end payment flow between two agents using the **A2A x402 Payment Protocol Extension**. It serves as a reference implementation for developers looking to add payment capabilities to their own agents.

The demo consists of two main components:
1.  A **Client Agent** that acts as an orchestrator, delegating tasks and handling the user-facing interaction.
2.  A **Merchant Server** that hosts a specialized agent capable of selling items and processing payments using the x402 protocol.

The reusable, core logic for the x402 protocol is encapsulated in the `x402_a2a` Python library, located in the `python/` directory of the parent repository.

## How to Run the Demo

### Prerequisites
- Python 3.13+
- `uv` (for environment and package management)
- Google API key (you can create one [here](https://ai.google.dev/gemini-api/docs/api-key))
- **For real payments**: CDP API credentials (optional, see Configuration section)

### 1. Setup the Environment
First, sync the virtual environment to install all necessary dependencies, including the local `x402_a2a` library in editable mode.

Run this command from the root of the `a2a-x402` repository:
```bash
uv sync --directory=examples/python/adk-demo
```

### 2. Configuration
Create a `.env` file in the `examples/python/adk-demo/` directory with the following variables:

```bash
# Required
GOOGLE_API_KEY=your_api_key_here

# Optional - Model Configuration
GEMINI_MODEL=gemini-1.5-flash  # Default: gemini-1.5-flash

# Optional - Facilitator Configuration
USE_MOCK_FACILITATOR=false     # Default: true (uses mock facilitator)
USE_MAINNET=false              # Default: false (uses testnet)

# Optional - For Real Payments (when USE_MOCK_FACILITATOR=false)
CDP_API_KEY_ID=your_cdp_key_id
CDP_API_KEY_SECRET=your_cdp_secret

# Optional - Wallet Configuration
CLIENT_PRIVATE_KEY=your_private_key_here
MERCHANT_WALLET_ADDRESS=your_merchant_wallet_address
```

> **Warning:** Do not hardcode or commit your API keys. The `.env` file is already included in `.gitignore`.

### 3. Start the Merchant Agent Server
The merchant server hosts the agent that sells products.

Run this command from the root of the `a2a-x402` repository:
```bash
uv --directory=examples/python/adk-demo run server
```
You should see logs indicating the server is running, typically on `localhost:10000`.

### 4. Start the Client Agent & Web UI
The client agent is an orchestrator that communicates with the merchant. The ADK provides a web interface to interact with it.

Run this command from the root of the `a2a-x402` repository:
```bash
uv --directory=examples/python/adk-demo run adk web --port=8000
```
This will start the ADK web server, usually on `localhost:8000`. Open this URL in your browser to interact with the client agent and start the purchase flow.

## Architectural Flow

The demo showcases a clean separation of concerns between the agent's business logic and the payment protocol logic.

1.  **Merchant-Side (Server):**
    - The `AdkMerchantAgent` contains the core business logic (e.g., providing product details). When payment is required, it doesn't handle any payment logic itself. Instead, it raises a `x402PaymentRequiredException`.
    - The `x402ServerExecutor` is a wrapper that intercepts this exception. It's responsible for all the server-side protocol logic: creating the `payment-required` response, receiving the client's signed payload, verifying it, and settling it.
    - This executor is "injected" in `routes.py`, wrapping the core `ADKAgentExecutor`.

2.  **Client-Side (`ClientAgent`):**
    - The `ClientAgent` acts as the user's proxy. Its `send_message` tool handles all communication.
    - When it receives a `payment-required` response from the merchant, it now prompts the user for confirmation.
    - Upon user confirmation, it calls its injected **Wallet** to sign the payment details.
    - It then uses the `x402Utils` from the core library to construct a valid `payment-submitted` message and sends it back to the merchant to finalize the purchase.

## Pluggable Components

A key design goal of this demo is to show how core components can be swapped out with real implementations.

### Facilitator
The `x402MerchantExecutor` requires a facilitator to verify and settle payments. The facilitator choice is controlled by the `USE_MOCK_FACILITATOR` environment variable (defaults to "true").

- When `USE_MOCK_FACILITATOR=true` (default), it uses a `MockFacilitator` (`mock_facilitator.py`) which approves all valid transactions, allowing you to test the payment flow without real transactions.
- When `USE_MOCK_FACILITATOR=false`, it uses a real `FacilitatorClient` with a provided `FacilitatorConfig` to process actual onchain transactions.

To use a real payment processor, set `USE_MOCK_FACILITATOR=false` and provide a valid `FacilitatorConfig`.

### Wallet
The `ClientAgent` does not handle signing directly. Instead, it depends on a `Wallet` interface (`wallet.py`). This makes the signing mechanism fully pluggable.

In the demo, we inject a `MockLocalWallet` which signs transactions using a private key from the `CLIENT_PRIVATE_KEY` environment variable. To connect to a real system, a developer could implement:
- A wallet that connects to a browser extension like MetaMask.
- A wallet that calls out to a secure MPC (Multi-Party Computation) service.
- A wallet that communicates with a hardware signing device.

This architecture ensures that the agent's orchestration logic remains completely separate from the specifics of payment signing.

## Recent Updates

This demo has been enhanced with the following improvements:

- **Real Facilitator Integration**: Support for both mock and real x402 facilitators
- **Environment Variable Configuration**: All settings configurable via `.env` file
- **Model Flexibility**: Configurable Gemini model selection (defaults to gemini-1.5-flash)
- **Testnet/Mainnet Support**: Easy switching between testnet and mainnet
- **Improved Architecture**: Cleaner separation of facilitator configuration logic
- **Testing-Friendly Pricing**: Reduced product prices for easier testing with limited funds
