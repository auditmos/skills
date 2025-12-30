# Auditmos Security Skills

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![Skills](https://img.shields.io/badge/Skills-14-brightgreen.svg)](#available-skills)

> 🔒 A comprehensive collection of **14 ready-to-use smart contract security audit skills** for Claude. Transform Claude into your AI security auditor capable of executing complex multi-step security analysis workflows across DeFi protocols, token standards, and blockchain applications.

These skills enable Claude to seamlessly audit smart contracts for critical security vulnerabilities across multiple attack vectors:
- 🔐 **Access Control & State Validation** - Access control bypasses, input validation, state consistency
- 💰 **DeFi Protocol Security** - Lending, liquidation, staking, CLM (Concentrated Liquidity Management), auction mechanisms
- 🧮 **Mathematical Precision** - Arithmetic errors, decimal precision, rounding issues, overflow/underflow
- 🔄 **Reentrancy Attacks** - Reentrancy vulnerabilities, cross-function attacks, external call patterns
- 📝 **Signature Security** - EIP-712 vulnerabilities, ecrecover misuse, replay attacks, signature malleability
- 💱 **DEX Integration Security** - Slippage protection, MEV vulnerabilities, sandwich attacks, oracle manipulation
- ⚖️ **Liquidation Security** - Unfair liquidation, DoS attacks, calculation errors, economic exploits
- 🎯 **Oracle Security** - Price manipulation, stale data, oracle dependency risks

**Transform Claude into an 'AI Security Auditor' on your desktop!**

> ⭐ **If you find this repository useful**, please consider giving it a star! It helps others discover these tools and encourages us to continue maintaining and expanding this collection.

---

## 📦 What's Included

This repository provides **14 specialized security audit skills** organized into the following categories:

- **DeFi Protocol Audits** - Lending protocols, liquidation mechanisms, staking contracts, CLM systems, auction protocols
- **Mathematical Security** - Precision errors, rounding issues, arithmetic vulnerabilities
- **Access Control** - State validation, input validation, access control bypasses
- **Signature & Authentication** - EIP-712, ecrecover, replay attacks
- **DEX Integration** - Slippage protection, MEV vulnerabilities, sandwich attacks
- **Oracle Security** - Price feed manipulation, stale data vulnerabilities
- **Reentrancy Protection** - Cross-function attacks, external call patterns

Each skill includes:
- ✅ Comprehensive documentation (`SKILL.md`)
- ✅ Vulnerability checklists (`checklist.md`)
- ✅ Practical code examples (`example.md`)
- ✅ Reference materials (`reference.md`)
- ✅ Report templates (`templates/report-template.md`)

---

## 📋 Table of Contents

- [What's Included](#whats-included)
- [Why Use This?](#why-use-this)
- [Getting Started](#getting-started)
- [Quick Examples](#quick-examples)
- [Available Skills](#available-skills)
- [How It Works](#how-it-works)
- [Contributing](#contributing)
- [Troubleshooting](#troubleshooting)
- [FAQ](#faq)
- [Support](#support)
- [License](#license)

---

## 🚀 Why Use This?

### ⚡ **Accelerate Your Security Audits**
- **Save Days of Work** - Skip manual vulnerability pattern research and checklist compilation
- **Production-Ready Checklists** - Tested, validated security patterns following industry best practices
- **Multi-Step Workflows** - Execute complex security analysis pipelines with a single prompt

### 🎯 **Comprehensive Coverage**
- **14 Specialized Skills** - Extensive coverage across all major smart contract attack vectors
- **Industry-Standard Patterns** - Based on real-world vulnerabilities and security research
- **Automated Discovery** - Skills auto-trigger based on code patterns and request keywords

### 🔧 **Easy Integration**
- **One-Click Setup** - Install via Claude Code
- **Automatic Discovery** - Claude automatically finds and uses relevant skills
- **Well Documented** - Each skill includes examples, checklists, and reference materials

### 🌟 **Maintained & Supported**
- **Regular Updates** - Continuously maintained and expanded
- **Community Driven** - Open source with active community contributions
- **Professional Quality** - Used in production security audits

---

## 🎯 Getting Started

### 🖥️ Claude Code (Recommended)

**Step 1: Install Claude Code**

**macOS:**
```bash
curl -fsSL https://claude.ai/install.sh | bash
```

**Windows:**
```powershell
irm https://claude.ai/install.ps1 | iex
```

**Step 2: Register the Marketplace**

```bash
/plugin marketplace add auditmos/skills
```

**Step 3: Install Skills**

1. Open Claude Code
2. Select **Browse and install plugins**
3. Choose **auditmos-agent-skills**
4. Select **smart-contract-security-skills**
5. Click **Install now**

**That's it!** Claude will automatically use the appropriate skills when you describe your security audit tasks.

---

## 💡 Quick Examples

### For Users:

Simply describe what you want audited:

- **"Audit this DEX integration"** → auto-triggers `audit-slippage`
- **"Check for precision errors"** → auto-triggers `audit-math-precision`
- **"Review signature security"** → auto-triggers `audit-signature`
- **"Complete security audit"** → triggers all relevant skills
- **"Audit this lending protocol"** → auto-triggers `audit-lending`
- **"Check liquidation mechanism"** → auto-triggers `audit-liquidation`
- **"Review reentrancy protection"** → auto-triggers `audit-reentrancy`

### Example Workflow:

```
User: "I need a security audit of this Uniswap V3 integration contract"

Claude: "I'm using the audit-slippage skill to analyze this contract for 
slippage protection and MEV vulnerabilities..."

[Claude automatically runs through the checklist, identifies vulnerabilities, 
and generates a comprehensive security report]
```

**Skills auto-discover based on code patterns and request keywords. No manual invocation needed.**

---

## 📚 Available Skills

### DeFi Protocol Security

- **audit-lending** - Lending protocol vulnerabilities, interest rate manipulation, collateral management
- **audit-liquidation** - Liquidation mechanism security, economic exploits, incentive misalignment
- **audit-liquidation-calculation** - Liquidation calculation errors, precision issues, edge cases
- **audit-liquidation-dos** - Liquidation DoS attacks, gas griefing, front-running protection
- **audit-unfair-liquidation** - Unfair liquidation attacks, MEV extraction, economic manipulation
- **audit-staking** - Staking contract security, reward distribution, slashing mechanisms
- **audit-clm** - Concentrated Liquidity Management security, position management, fee collection
- **audit-auction** - Auction mechanism security, bid validation, economic exploits

### Mathematical & Precision Security

- **audit-math-precision** - Arithmetic errors, decimal precision, rounding issues, overflow/underflow

### Access Control & Validation

- **audit-state-validation** - Access control bypasses, input validation, state consistency checks

### Signature & Authentication

- **audit-signature** - EIP-712 vulnerabilities, ecrecover misuse, replay attacks, signature malleability

### DEX Integration Security

- **audit-slippage** - Slippage protection, MEV vulnerabilities, sandwich attacks, deadline validation

### Oracle Security

- **audit-oracle** - Price feed manipulation, stale data vulnerabilities, oracle dependency risks

### Reentrancy Protection

- **audit-reentrancy** - Reentrancy vulnerabilities, cross-function attacks, external call patterns

---

## 🔍 How It Works

### Automatic Skill Discovery

Skills automatically trigger based on:

1. **Code Patterns** - Claude analyzes your code and identifies relevant security concerns
2. **Request Keywords** - Keywords in your request trigger appropriate skills
3. **Context Awareness** - Skills work together when multiple vulnerabilities are present

### Audit Workflow

Each skill follows a structured workflow:

1. **Pattern Identification** - Searches codebase for relevant patterns
2. **Vulnerability Checking** - Compares against comprehensive checklists
3. **Exploitability Analysis** - Assesses severity and exploitability
4. **Report Generation** - Creates detailed security reports with PoCs

### Skill Structure

Each skill includes:

- **SKILL.md** - Main documentation and workflow instructions
- **checklist.md** - Comprehensive vulnerability checklist
- **example.md** - Code examples and patterns
- **reference.md** - Reference materials and attack vectors
- **templates/report-template.md** - Standardized report format

---

## 🤝 Contributing

We welcome contributions to expand and improve this security audit skills repository!

### Ways to Contribute

✨ **Add New Skills**
- Create skills for additional attack vectors or protocol types
- Add integrations for new DeFi protocols or standards

📚 **Improve Existing Skills**
- Enhance documentation with more examples and use cases
- Add new vulnerability patterns and attack vectors
- Improve code examples and checklists
- Fix bugs or update outdated information

🐛 **Report Issues**
- Submit bug reports with detailed reproduction steps
- Suggest improvements or new features

### How to Contribute

1. **Fork** the repository
2. **Create** a feature branch (`git checkout -b feature/amazing-skill`)
3. **Follow** the existing directory structure and documentation patterns
4. **Ensure** all new skills include comprehensive `SKILL.md` files
5. **Test** your examples and workflows thoroughly
6. **Commit** your changes (`git commit -m 'Add amazing skill'`)
7. **Push** to your branch (`git push origin feature/amazing-skill`)
8. **Submit** a pull request with a clear description of your changes

### Contribution Guidelines

✅ Maintain consistency with existing skill documentation format  
✅ Include practical, working examples in all contributions  
✅ Ensure all code examples are tested and functional  
✅ Follow security best practices in examples and workflows  
✅ Update relevant documentation when adding new capabilities  
✅ Provide clear comments and docstrings in code  
✅ Include references to security research and CVE reports

---

## 🔧 Troubleshooting

### Common Issues

**Problem: Skills not loading in Claude Code**
- Solution: Ensure you've installed the latest version of Claude Code
- Try reinstalling the plugin

**Problem: Skills not triggering automatically**
- Solution: Be more specific in your request (e.g., "audit for slippage vulnerabilities")
- Check that relevant code patterns exist in your codebase

**Problem: Missing vulnerability patterns**
- Solution: Check the specific `checklist.md` file for complete vulnerability list
- Review `reference.md` for additional attack vectors

**Problem: Outdated examples**
- Solution: Report the issue via GitHub Issues
- Check the official security research for updated patterns

---

## ❓ FAQ

### General Questions

**Q: Is this free to use?**  
A: Yes! This project is MIT licensed, allowing free use for any purpose including commercial projects.

**Q: Can I use this for commercial security audits?**  
A: Absolutely! The MIT License allows both commercial and noncommercial use without restrictions.

**Q: How accurate are these security checks?**  
A: Skills are based on industry-standard security patterns and real-world vulnerabilities. However, they should complement, not replace, professional security audits.

**Q: How often is this updated?**  
A: We regularly update skills to reflect new attack vectors and security research. Major updates are announced in release notes.

**Q: Can I use this with other AI models?**  
A: The skills are optimized for Claude but can be adapted for other models that support similar skill/plugin systems.

### Installation & Setup

**Q: Do I need all the skills installed?**  
A: No! Skills are automatically selected based on your code and requests. You can install the full collection for comprehensive coverage.

**Q: What if a skill doesn't work?**  
A: First check the [Troubleshooting](#troubleshooting) section. If the issue persists, file an issue on GitHub with detailed reproduction steps.

**Q: Do the skills work offline?**  
A: Yes! Skills analyze your local codebase and don't require internet access (unless accessing external references).

### Contributing

**Q: Can I contribute my own skills?**  
A: Absolutely! We welcome contributions. See the [Contributing](#contributing) section for guidelines and best practices.

**Q: How do I report bugs or suggest features?**  
A: Open an issue on GitHub with a clear description. For bugs, include reproduction steps and expected vs actual behavior.

---

## 💬 Support

Need help? Here's how to get support:

- 📖 **Documentation**: Check the relevant `SKILL.md` and `reference.md` files
- 🐛 **Bug Reports**: [Open an issue](https://github.com/[your-repo]/issues)
- 💡 **Feature Requests**: [Submit a feature request](https://github.com/[your-repo]/issues/new)
- 📧 **Email**: tom@auditmos.com

---

## 📄 License

This project is licensed under the **MIT License**.

**Copyright © 2025 Auditmos**

### Key Points:
- ✅ **Free for any use** (commercial and noncommercial)
- ✅ **Open source** - modify, distribute, and use freely
- ✅ **Permissive** - minimal restrictions on reuse
- ⚠️ **No warranty** - provided "as is" without warranty of any kind

See [LICENSE](LICENSE) for full terms.

---

## 🙏 Acknowledgments

Special thanks to the security research community and all contributors who help make smart contract security more accessible through AI-powered auditing tools.
