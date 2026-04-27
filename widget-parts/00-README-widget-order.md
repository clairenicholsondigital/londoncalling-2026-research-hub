# Widget split map for `main.html`

These files are intentionally standalone snippets so you can stack them as independent HTML widgets in your event platform.

Suggested top-to-bottom order:

1. `01-global-styles.html`
2. `02-root-and-nav-header.html`
3. `03-area-button-bar.html`
4. `04-hub-header.html`
5. `05-tab-navigation.html`
6. `06-panel-sessions.html`
7. `07-panel-posters.html`
8. `08-panel-documents.html`
9. `09-panel-videos.html`
10. `10-panel-secret-cinema.html`
11. `11-modal-viewer.html`
12. `12-close-hub-and-root.html`
13. Either `13-script-full.html` OR the script chunks below:
    - `14-script-bootstrap-and-config.html`
    - `15-script-utils-and-modal.html`
    - `16-script-data-and-renderers.html`
    - `17-script-loading-countdown-and-polish.html`
    - `18-script-init.html`

Notes:
- `13-script-full.html` is the complete original script block.
- `14` to `18` are the same script split by responsibility so you can place logic blocks separately.
- Imports are intentionally not handled, per your request.
