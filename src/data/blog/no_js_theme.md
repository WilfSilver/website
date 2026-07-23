---
title: The search for a no-JS theme toggle
author: Wilf Silver
pubDatetime: 2026-07-23T20:10:00Z
slug: nojs_theme
featured: true
draft: false
tags:
  - frontend
  - javascript
  - css
  - tailwindcss
description: With recent powerful CSS features, we are getting closer to the possibility of zero JS for a theme toggle. This runs through my current preferred method.
---

I have recently gone through a bit of an obsession of reducing the required JavaScript in websites as much as possible, from using the new `command-for` attributes for modals and dropdown menus to having that theme button. My obsession with having a theme button on websites comes from getting eye strain from looking at blazingly white websites while at a full-time job -- which let me install the [dark reader extension](https://darkreader.org/). However, modern websites, such as [tailwindcss](https://tailwindcss.com/), have started removing this option in-place of automatic theme based on what you tell your browser, and then it looks terrible with dark reader extension. On the surface, this is great, no JS at all, and you get the choice you want. The problem? I use [Librewolf](https://www.librewolf.net/), which by default does not let you change this, as it can be used for fingerprinting. We will ignore the fact that dark reader is also a fingerprint point, especially when paired with "preferring" a light theme on websites. On top of this, I also use the [NoScript extension](https://noscript.net/) to block any unnecessary third-party website scripts to reduce tracking.

This all has made me extremely interested in finding a fully no-JS friendly alternative to allowing people to choose (and save) their preferred theme for a website. There are several problems:

1. Usually, user-defined choices require altering the root HTML element (as you can't rely on modals). So how do you edit this without JavaScript
1. You want to save the preferred theme, so when the person revisits the website, they don't have to click another button, this is usually done with cookies, but that requires JavaScript...
1. By default, you want to make sure you're using the person's preferred theme, this is usually done through JS.
1. How do you allow the person to reset to "automatic" -- Plenty of websites including just have a basic "toggle" where you can never go back without clearing cookies

So now we have this, the examples will be utilising tailwindcss as that makes it easy to have multiple themes. This use of tailwindcss, does come with a minor disadvantage, we are restricted to using their system for theming, which is done on a per-element basis. With modern CSS, you can just have a root element define the theming by looking for a state of a child element with `has`. E.g.

```css
html {
  /* Light theme */
}

@media (prefers-color-scheme: light) {
  html:has(button#themeswitch:checked) {
    /* Dark theme definitions */
  }
}

@media (prefers-color-scheme: dark) {
  html:has(button#themeswitch:not(:checked)) {
    /* Dark theme definitions */
  }
}
```

This doesn't solve the saving problem, nor the "automatic reset", but it is a basic example that is fully no JS. I also personally don't like the duplication as it feels unneeded, but you will see the later solutions all mostly require at least some duplication (but handled by tailwindcss).

## Minimal JS

My current preferred method of JS (as a basic toggle button) is using as minimal JS as possible, and then defaulting to the browser's preference if JS is disabled. This can be done through something like:

```html
<button
  id="toggle-theme"
  class="hover:text-white hover:bg-amber-700 px-2 py-2 rounded-md active:bg-amber-900 active:ring-4 focus:ring-4 ring-amber-500 transition-all cursor-pointer noscript:hidden"
  type="button"
  aria-label="Switch between light and dark mode"
  title="Please enable JavaScript for this functionality."
>
  <!-- Icons taken from Phosphor icon pack: https://phosphoricons.com -->
  <svg class="hidden aspect-square size-8 dark:block" xmlns="http://www.w3.org/2000/svg" width="32" height="32" fill="#000000" viewBox="0 0 256 256"><path d="M120,40V16a8,8,0,0,1,16,0V40a8,8,0,0,1-16,0Zm72,88a64,64,0,1,1-64-64A64.07,64.07,0,0,1,192,128Zm-16,0a48,48,0,1,0-48,48A48.05,48.05,0,0,0,176,128ZM58.34,69.66A8,8,0,0,0,69.66,58.34l-16-16A8,8,0,0,0,42.34,53.66Zm0,116.68-16,16a8,8,0,0,0,11.32,11.32l16-16a8,8,0,0,0-11.32-11.32ZM192,72a8,8,0,0,0,5.66-2.34l16-16a8,8,0,0,0-11.32-11.32l-16,16A8,8,0,0,0,192,72Zm5.66,114.34a8,8,0,0,0-11.32,11.32l16,16a8,8,0,0,0,11.32-11.32ZM48,128a8,8,0,0,0-8-8H16a8,8,0,0,0,0,16H40A8,8,0,0,0,48,128Zm80,80a8,8,0,0,0-8,8v24a8,8,0,0,0,16,0V216A8,8,0,0,0,128,208Zm112-88H216a8,8,0,0,0,0,16h24a8,8,0,0,0,0-16Z"></path></svg>
  <svg class="aspect-square size-8 dark:hidden" xmlns="http://www.w3.org/2000/svg" width="32" height="32" fill="#000000" viewBox="0 0 256 256"><path d="M233.54,142.23a8,8,0,0,0-8-2,88.08,88.08,0,0,1-109.8-109.8,8,8,0,0,0-10-10,104.84,104.84,0,0,0-52.91,37A104,104,0,0,0,136,224a103.09,103.09,0,0,0,62.52-20.88,104.84,104.84,0,0,0,37-52.91A8,8,0,0,0,233.54,142.23ZM188.9,190.34A88,88,0,0,1,65.66,67.11a89,89,0,0,1,31.4-26A106,106,0,0,0,96,56,104.11,104.11,0,0,0,200,160a106,106,0,0,0,14.92-1.06A89,89,0,0,1,188.9,190.34Z"></path></svg>
</button>

<!-- ... -->

<script>
  const btn = document.getElementById("toggle-theme");

  // Initialises dark mode based on cookies, or if not present, users
  // browser preference for the theme.
  let isDarkMode = document.documentElement.classList.toggle(
    "dark",
    localStorage.theme === "dark" ||
      (!("theme" in localStorage) &&
        window.matchMedia("(prefers-color-scheme: dark)").matches),
  );

  /**
   * Updates the aria-label and title on the theme button to more accurately
   * resemble the action being performed
   */
  function updateThemeBtn() {
    if (!btn) return;

    const infoText = `Switch to ${isDarkMode ? "light" : "dark"} theme.`;
    btn.setAttribute("aria-label", infoText);
    btn.title = infoText;
  }

  /**
   * Toggles between light and dark theme, updating cookies and any required titles.
   */
  function toggleTheme() {
    isDarkMode = document.documentElement.classList.toggle("dark");
    localStorage.theme = isDarkMode ? "dark" : "light";

    updateThemeBtn();
  }

  if (btn) {
    updateThemeBtn();
    btn.onclick = toggleTheme;
  }
</script>
```

The tailwind config is as such:

```css
@import "tailwindcss";

@custom-variant dark {
  &:where(.dark, .dark *) {
    @slot;
  }

  /* If JavaScript is disabled, we can just show the persons preferred theme */
  @media (scripting: none) and (prefers-color-scheme: dark) {
    @slot;
  }
}
```

This uses cookies to save the preferred theme and uses CSS to show the correct icon on the button, which means the JS is only required to add the "dark" class to the document element, update cookies and then update the title and aria label (which is optional). This is cool, but I think I can take this further.

## Pure CSS switching

Now with the release of `:has` and parent selectors, you can now remove the required `document.documentElement.classList.toggle("dark")`. In place we can use the CSS:

```css
@custom-variant dark {
  html:has(input#dark-theme:checked) & {
    @slot;
  }
  /* ... */
}
```

This works by only applying the dark styles when a parent matching `html:has(input#dark-theme:checked)`, which is matched when the root element `html` has the `#dark-theme` input checked. This is cool, but we can go further, by swapping to a select element we can add an option for `automatic` which is selected by default:

```html
<select id="themes">
  <option value="auto" selected>Automatic</option>
  <option value="light">Light</option>
  <option value="dark">Dark</option>
</select>
```

Which the following tailwindcss config:

```css
@import "tailwindcss";

@custom-variant dark {
  html:has(select#themes > option[value="dark"]:checked) & {
    @slot;
  }

  html:has(select#themes > option[value="auto"]:checked) & {
    @media (prefers-color-scheme: dark) {
      @slot;
    }
  }
}
```

Notice here we are now matching if the `option` is "checked", which in modern browsers basically translates to if it is selected.

This also comes with the partial benefit that on reload, your selection is saved (on Firefox only). But when you close and reopen the page, it is not saved, which is an issue!

### Adding back saving

This is where everything starts to fall apart. As noted previously, Firefox somewhat saves your preference in the current tab if you reload it, but sadly there is no current way to extend this to saving choices across tabs. Therefore, this leads to the simple solution of:

```html
<script>
  const themes = document.getElementById("themes");

  if (themes) {
    themes.value = localStorage.theme ?? "auto";

    themes.onchange = () => {
      localStorage.theme = themes.value;
    }
  }
</script>
```

This does have the slight disadvantage of having a slight flicker on startup for fast systems, but I don't believe that it is a dealbreaker.

## Conclusion

It is amazing the progress of no-JS systems with the introduction of CSS 5, but it is still disappointing that we cannot get any saving or caching behaviour for specific inputs, allowing for consistent behaviour across all pages (it would be even better if these synced automatically across multiple tabs). I remember hearing about a no-JS working group within one of the standards organisations, which would be good to progress this form. From a privacy perspective, I believe that JavaScript has become such a monolithic creature that it would be good to have some decent alternatives which doesn't enable instant fingerprinting and overbearingly heavy pages, though these can also be solved by better API design, sandboxing and permissions requests.

Having said all this, I should note that using JavaScript in websites is not the end of the world, it enables much more cool dynamic, optimised websites. This was mostly just a fun experiment and research project for me, and I would love to see more possibilities with even more no-JS websites.
