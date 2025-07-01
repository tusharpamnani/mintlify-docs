# Shardeum JSON-RPC Documentation

This project provides comprehensive documentation for the Shardeum JSON-RPC API, including guides, method references, and usage examples. It is built using [Mintlify Docs](https://mintlify.com/).

## 📚 Features
- Guides for getting started and advanced usage
- Reference for Shardeum and Ethereum-compatible JSON-RPC methods
- Example code and step-by-step explanations
- Custom branding and logo

## 🚀 Getting Started

### Prerequisites
- [Node.js](https://nodejs.org/) (v14 or higher recommended)
- [npm](https://www.npmjs.com/)
- [Mintlify CLI](https://www.npmjs.com/package/mint)

### Installation
1. Clone this repository:
   ```bash
   git clone https://github.com/tusharpamnani/mintlify-docs
   cd mintlify-docs
   ```
2. Install the Mintlify CLI globally (if not already installed):
   ```bash
   npm i -g mint
   ```

### Local Development
To preview documentation changes locally:
```bash
mint dev
```
This will start a local server. Open the provided URL in your browser to view the docs.

## 🛠️ Customization
- **Logo:** Update `docs.json` and the `logo/` directory to change branding.
- **Colors:** Edit the `colors` section in `docs.json`.
- **Navigation:** Organize pages and groups in the `navigation` section of `docs.json`.
- **Content:** Edit or add `.mdx` and `.md` files in the root directory for new guides or references.

## 📦 Deployment
If using Mintlify's hosted platform, push changes to your default branch and they will be deployed automatically. For self-hosted or static export, refer to [Mintlify's documentation](https://docs.mintlify.com).

## 🤝 Contributing
Pull requests and issues are welcome! Please open an issue to discuss major changes first.

