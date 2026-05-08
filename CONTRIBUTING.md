# Contributing to Continuum.Markets

Thank you for your interest in contributing to Continuum.Markets! This document provides guidelines for contributing to the project.

## Code of Conduct

- Be respectful and inclusive
- Focus on constructive feedback
- Help newcomers get started
- Report security issues privately (security@continuum.markets)

## Getting Started

1. Fork the repository
2. Create a feature branch (`git checkout -b feature/amazing-feature`)
3. Make your changes
4. Test thoroughly (see Development Guide)
5. Commit with clear messages
6. Push to your fork
7. Open a Pull Request

## Development Setup

See [DEVELOPMENT.md](./DEVELOPMENT.md) for detailed setup instructions.

```bash
# Quick start
git clone https://github.com/your-fork/continuum.git
cd continuum
pnpm install
cd frontend && pnpm install
```

## Pull Request Process

1. **Update Documentation**: If you change APIs or add features, update relevant docs
2. **Write Tests**: Add unit tests for Rust code, integration tests for programs
3. **Format Code**:
   ```bash
   # Rust
   cargo fmt
   
   # TypeScript
   cd frontend && pnpm lint --fix
   ```
4. **Check Types**: `pnpm type-check` in frontend
5. **Build Successfully**: Both programs and frontend should build without errors
6. **Describe Changes**: Write a clear PR description explaining what and why

## Commit Messages

Use conventional commits:

```
feat: add BRENT oil market
fix: correct NAV calculation for shorts
docs: update deployment guide
test: add funding rate tests
refactor: simplify AMM swap logic
```

## What to Contribute

### High Priority
- Additional unit tests for programs
- Integration tests with multiple markets
- Frontend error handling improvements
- Performance optimizations
- Documentation improvements

### Ideas
- New market listings (with oracle support)
- UI/UX enhancements
- Mobile responsiveness improvements
- Accessibility (a11y) improvements
- i18n support

### Not Accepted
- Breaking changes without discussion
- Code without tests
- Features that compromise security
- Changes that increase centralization

## Program Development

When contributing to Solana programs:

1. **Security First**: All changes must pass security review
2. **Use Anchor Constraints**: Prefer declarative constraints over runtime checks
3. **Checked Math**: Always use `checked_*` operations
4. **Error Handling**: Use custom error codes, not panic!
5. **Gas Optimization**: Minimize compute units where possible

### Example
```rust
// Good
let result = a.checked_mul(b).ok_or(ErrorCode::Overflow)?;

// Bad
let result = a * b; // Can panic!
```

## Frontend Development

1. **TypeScript Strict Mode**: All new code must pass strict type checking
2. **Component Structure**: Use Shadcn/UI patterns for consistency
3. **Accessibility**: Add ARIA labels, keyboard navigation
4. **Mobile-First**: Test on mobile viewports
5. **Error States**: Handle loading, error, and empty states

### Example
```tsx
// Good
const { data, isLoading, error } = useQuery(['markets'], fetchMarkets);

if (isLoading) return <Skeleton />;
if (error) return <ErrorMessage error={error} />;
if (!data) return <EmptyState />;

// Bad
const data = useQuery(['markets'], fetchMarkets).data;
return <div>{data.map(...)}</div>; // Can crash!
```

## Testing Guidelines

### Rust Tests
```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_nav_calculation() {
        let oracle_price = 5000;
        let initial_l = 5000;
        let initial_s = 5000;
        
        let s_nav = calculate_short_nav(oracle_price, initial_l, initial_s);
        assert_eq!(s_nav, 5000);
    }
}
```

### Integration Tests
```typescript
describe("Mint/Redeem", () => {
  it("should mint long tokens", async () => {
    const amount = 1000;
    const tx = await program.methods
      .mintTokens(amount, { long: {} })
      .rpc();
    
    expect(tx).toBeDefined();
    // Verify token balance increased
  });
});
```

## Documentation

When adding features, update:
- README.md (if user-facing)
- ARCHITECTURE.md (if changing system design)
- DEVELOPMENT.md (if changing dev workflow)
- Inline code comments (for complex logic)

## Security

**CRITICAL**: If you discover a security vulnerability:

1. **DO NOT** open a public issue
2. Email security@continuum.markets with details
3. Wait for response before disclosing
4. We aim to respond within 24 hours

Eligible for bug bounty (up to $50k for critical issues).

## Review Process

1. Automated checks must pass (CI/CD)
2. At least one maintainer approval required
3. Security review for program changes
4. Final approval from core team

## Community

- **Discord**: [discord.gg/continuum](https://discord.gg/continuum)
  - #dev-chat - General development discussion
  - #dev-help - Get help with contributions
  - #proposals - Discuss feature ideas

- **GitHub Discussions**: For longer-form discussions

- **Twitter**: [@ContinuumMkts](https://twitter.com/ContinuumMkts) - Announcements

## Recognition

Contributors will be:
- Listed in CONTRIBUTORS.md
- Mentioned in release notes
- Eligible for CNTM token rewards (governance)
- Invited to contributor calls

## License

By contributing, you agree that your contributions will be licensed under the MIT License.

---

**Thank you for helping build the future of decentralized finance on Solana!** 🚀





