Reentrancy Protection
* [ ] State changes before external calls (CEI pattern)
* [ ] NonReentrant modifiers on vulnerable functions
* [ ] No assumptions about token transfer behavior
* [ ] Cross-function reentrancy considered
* [ ] Read-only reentrancy risks evaluated

Token Compatibility
* [ ] Fee-on-transfer tokens handled correctly
* [ ] Rebasing tokens accounted for
* [ ] Tokens with callbacks (ERC777) considered
* [ ] Zero transfer reverting tokens handled
* [ ] Pausable tokens won't brick protocol
* [ ] Token decimals properly scaled
* [ ] Deflationary/inflationary tokens supported

Access Control
* [ ] Critical functions have appropriate modifiers
* [ ] Two-step ownership transfer implemented
* [ ] Role-based permissions properly segregated
* [ ] Emergency pause functionality included
* [ ] Time delays for critical operations