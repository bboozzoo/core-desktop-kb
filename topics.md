# discussion topics

## desktop launch

See `desktop-launch.md`.

- snap routine desktop-launch vs dedicated helper binary
  - within snapd snap? too big?
  - maybe core26-desktop
  - we still need to patch Exec= in desktop files to call it
    - how to determine when to do it? base core26-desktop base? is it available in wrappers/desktop?
- what happens with stdio?
  - not on the terminal, not in journal - breaks expectations?
  - apps launched using systemd-run
  
 - [ ] check for existing usages of desktop-launch
 - [ ] potential maintainability problem - what if we update snapd and change the format

- core26-desktop base going away?

