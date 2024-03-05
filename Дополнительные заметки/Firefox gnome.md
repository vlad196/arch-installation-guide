Так как Epiphany плохо у меня работает, проще применить тему к Firefox
>[!info]
>Надо до разобраться с конфигами

тема:
```bash
curl -s -o- https://raw.githubusercontent.com/rafaelmardojai/firefox-gnome-theme/master/scripts/install-by-curl.sh | bash
```

### [Required Firefox preferences](https://github.com/rafaelmardojai/firefox-gnome-theme#required-firefox-preferences)

We provide a **user.js** configuration file in `configuration/user.js` that enable some preferences required by this theme to work.

You should already have this file installed if you followed one of the installation methods, but in any case be sure this preferences are enabled under `about:config`:

- `toolkit.legacyUserProfileCustomizations.stylesheets`
    
    This preference is required to load the custom CSS in Firefox, otherwise the theme wouldn't work.
    
- `svg.context-properties.content.enabled`
    
    This preference is required to recolor the icons, otherwise you will get black icons everywhere.
    

> For other non essential preferences checkout `configuration/user.js`.


## [Enabling optional features](https://github.com/rafaelmardojai/firefox-gnome-theme#enabling-optional-features)

Optional features can be enabled by creating new `boolean` preferences in `about:config`.

1. Go to the `about:config` page
2. Type the key of the feature you want to enable
3. Set it as a `boolean` and click on the add button
4. Restart Firefox

### [Features](https://github.com/rafaelmardojai/firefox-gnome-theme#features)

- **Hide single tab** `gnomeTheme.hideSingleTab`
    
    Hide the tab bar when only one tab is open.
    
    > **Note:** You should move the new tab button out of the tabbar or it will be hidden when there is only one tab. You can rearrange the toolbars doing a right-click on any toolbar and selecting "Customize Toolbar…".
    
- **Normal width tabs** `gnomeTheme.normalWidthTabs`
    
    Use normal width tabs as default Firefox.
    
- **Bookmarks toolbar under tabs** `gnomeTheme.bookmarksToolbarUnderTabs`
    
    Move Bookmarks toolbar under tabs.
    
- **Active tab contrast** `gnomeTheme.activeTabContrast`
    
    Add more contrast to the active tab.
    
- **Close only selected tabs** `gnomeTheme.closeOnlySelectedTabs`
    
    Show the close button on the selected tab only.
    
- **System icons** `gnomeTheme.systemIcons`
    
    Use system theme icons instead of Adwaita icons included by theme.
    
    > **Note:** This feature has a [known color bug](https://github.com/rafaelmardojai/firefox-gnome-theme#icons-color-broken-with-system-icons).
    
- **Symbolic tab icons** `gnomeTheme.symbolicTabIcons`
    
    Make all tab icons look kinda like symbolic icons.
    
- **Hide WebRTC indicator** `gnomeTheme.hideWebrtcIndicator`
    
    Hide redundant WebRTC indicator since GNOME provides their own privacy icons in the top right.
    
- **Hide unified extensions button** `gnomeTheme.hideUnifiedExtensions`
    
    Hide unified extensions button from the navbar, you can also use `extensions.unifiedExtensions.enabled` instead, which is only going to work till Firefox 111.
    
- **Drag window from headerbar buttons** `gnomeTheme.dragWindowHeaderbarButtons`
    
    Allow dragging the window from headerbar buttons.
    
    > **Note:** This feature is BUGGED. It can activate the button with unpleasant behavior.
    
- **Tabs as headerbar** `gnomeTheme.tabsAsHeaderbar`
    
    Place the tabs on the top of the window, and use the tabs bar to hold the window controls, like Firefox's standard tab bar.
    
    > **Note:** Enabling with `gnomeTheme.hideSingleTab` will replace the single tab with a title bar.
    
    ### [Extensions support](https://github.com/rafaelmardojai/firefox-gnome-theme#extensions-support)

We also have optional features to enable support for some Firefox extensions.

> **Be aware that extensions support are maintained by the community, so requests to support new extensions are not allowed and the included ones could get broken until someone shows up to fix them.**

- **Tab center reborn support** `gnomeTheme.extensions.tabCenterReborn`
    
    Enable the vertical tab trough the extension : [Tab Center Reborn](https://addons.mozilla.org/en-US/firefox/addon/tabcenter-reborn/).
    
    > **Note:** You also need to copy the contents of the file `configuration/extensions/tab-center-reborn.css` into the settings page of Tabcenter-reborn..  
    > **Note2:** You can also maintain the vertical tab always open with `gnomeTheme.extensions.tabCenterReborn.alwaysOpen`