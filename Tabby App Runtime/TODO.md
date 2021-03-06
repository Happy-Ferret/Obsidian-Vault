- [ ] Rename/Archive existing GitHub repository.
- [ ] Create new GitHub repository.
- [ ] Import LICENSE, README and necessary boilerplate.
- [ ] Test and expand with regular Firefox Nightly installation.
	- [ ] Add astilectron-esque JavaScript client library.
	- [ ] Run simple WAMP message pump based on https://github.com/gammazero/nexus
	- [ ] Create WAMP wrapper to make adding new functions as easy as on Go-Astilectron.
- [ ] Locate portable versions of Firefox/Firefox Nightly.
- [ ] Upload Firefox/Firefox Nightly to easily retrievable, central location.
- [ ] Design `version` file that tells provisioning tool what version of Firefox to download.
- [ ] Create provisioning tool to download Firefox release into app runtime directory.
- [ ] Create runtime to read an app folder's __package.json__ and launch `main` file entry.
- [ ] IF no __package.json__ was found, launch from `application.ini` instead.

- [ ] Tabby JS Library (Junior)
	- [ ] Remote module based on this: https://udn.realityripple.com/docs/Archive/Add-ons/Interaction_between_privileged_and_non-privileged_pages 
	- [ ] Preferences module
		- [ ] https://udn.realityripple.com/docs/Mozilla/Working_with_windows_in_chrome_code
	- [ ] Simple dialogs
	- [ ] CoreGraphics wrapper (for PDF and other rendering contexts)
	- [ ] `BrowserWindow`-esque API with window type CONSTs
		- [ ] [Electron BrowserWindow Documentation -- Look at Desktop type in Mac/Linux](https://www.electronjs.org/docs/latest/api/browser-window#new-browserwindowoptions)

- [ ] Tabby Runtime
	- [ ] Add light/dark theme command
	- [ ] Add light/dark theme switcher in window frame (see Visual Studio Installer)
		- [ ] ![[Pasted image 20220512125422.png]]
			- [ ] Make configurable through `BrowserWindow` options
			- [ ] [Example app](https://developer.mozilla.org/en-US/docs/Web/CSS/@media/prefers-color-scheme)
			- [ ] [Prefs](https://github.com/mozilla/gecko-dev/blob/e2e57ccf30c61eb0d41abde0e27006ccb8b88b08/browser/components/pocket/content/panels/js/style-guide/entry.js#L16)
			- [ ] 
		- [ ] CommandHandler ideas:
			- [ ] https://wiki.mozilla.org/XUL:Command_Line_Handling

- [ ] Tabby Runtime Package Manager
	- [ ] Part of regular Tabby command (codebase should be modular enough)
	- [ ] Use Git backend (GitHub org as a package registry, perhaps?)
	- [ ] Local backups to NAS or, better yet, tape.

- [ ] Tabby Bootstrap
	- [ ] Part of regular Tabby command (codebase should be modular enough)
	- [ ] XUL and Electron-like support
	- [ ] Ask for application name, author, website, version, description
	- [ ] `Update` command to update the fields (either sequentially or by declaring on commandline -- either interactive input or by typing new value like in `git commit -m`)

- [ ] Investigate [UXP](https://github.com/RealityRipple/UXP/) as an alternative to official Firefox host
