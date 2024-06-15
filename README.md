# Konkretnik Content Switcher for Ghost CMS
Ghost injection script for content switching. https://konkretnik.com

## Introduction

This repository provides a Content Switcher script designed specifically for Ghost CMS. The purpose of this script is to allow for easy content switching and text replacement without the need to modify the core Ghost code or themes. This approach ensures better maintenance and flexibility, allowing you to manage content dynamically across different pages.

## Features

- **Dynamic Content Switching:** Easily switch content blocks within specified areas.
- **Text Replacement:** Replace text globally within specified selectors or across the entire site.
- **No Theme Modification Required:** Works independently of the main Ghost themes, ensuring easy updates and maintenance.

## Version

Current version: 0.3.4

## Author

Bartosz Kulczyk  
Website: [https://konkretnik.com](https://konkretnik.com)  
License: MIT

## Installation

Follow these steps to integrate the Content Switcher into your Ghost CMS site.

### 1. Add CSS and JavaScript

Add the following CSS and JavaScript to your site. This can be done by editing the header of your Ghost theme or by injecting code via the Ghost Admin interface.

```html
<!--
Name: Konkretnik Content Switcher 
Author: Bartosz Kulczyk
Website: https://konkretnik.com
License: MIT
Version: 0.3.3
-->

<style>
  body>div.gh-site>main>div>article,
  #gh-head>div>nav {
    opacity: 0;
    animation: fadeIn .5s linear forwards;
  }
  
  @keyframes fadeIn {
    0%, 50% { opacity: 0; }
    100% { opacity: 1; }
  }
  
  .hidden { display: none; }
  .fade-in { opacity: 0; transition: opacity .5s; }
  .fade-in.visible { opacity: 1; }
  .contentBlock-content { display: none; }
  .contentBlock-content.visible { display: inline; }
  .active { font-weight: bold; }
</style>

<script>
  // Configuration for the content switcher and text replacements
  window.contentSwitcherConfig = {
    scanSelector: 'body>div.gh-site>main>div>article, #gh-head>div>nav',
    selectorReplacementTexts: {
      '<a href="https://konkretnik.com/">ChangeLang</a>': '||en||[[pl]]PL[[/pl]]||/en||||pl||[[en]]EN[[/en]]||/pl||',
      '<a href="https://konkretnik.com/">About</a>': '||en||<a href="https://konkretnik.com/">About me</a>||/en||||pl||<a href="https://konkretnik.com/">O mnie</a>||/pl||'
    },
    outsideSelectorReplacements: {
      en: {},
      pl: {
        'Subscribe': 'Subskrybuj',
        'Sign in': 'Zaloguj się',
        'Sign up': 'Załóż konto'
      }
    }
  };
</script>

<script>
  // Main Content Switcher Script
  document.addEventListener("DOMContentLoaded", () => {
    const { scanSelector, selectorReplacementTexts, outsideSelectorReplacements } = window.contentSwitcherConfig;
    const pageSpecificReplacements = { ...window.pageSpecificReplacements, ...selectorReplacementTexts };

    const findAndReplaceText = (content, replacements) => {
      for (const [search, replace] of Object.entries(replacements)) {
        const escapedSearch = search.replace(/[.*+?^${}()|[\]\\]/g, '\\$&');
        content = content.replace(new RegExp(escapedSearch, 'g'), replace);
      }
      return content;
    };

    const replaceContentBlocks = (content) => {
      const contentBlockPattern = /\|\|(\w+)\|\|([\s\S]*?)\|\|\/\1\|\|/g;
      const switchPattern = /\[\[(\w+)\]\]([\s\S]*?)\[\[\/\1\]\]/g;
      const contentBlocks = {};

      content = content
        .replace(contentBlockPattern, (match, contentBlock, text) => {
          contentBlocks[contentBlock] = contentBlocks[contentBlock] || [];
          contentBlocks[contentBlock].push(text);
          return `<span class="contentBlock-content" data-contentBlock="${contentBlock}">${text}</span>`;
        })
        .replace(switchPattern, (match, contentBlock, text) => `<a href="#" class="contentBlock-switch" data-contentBlock="${contentBlock}">${text}</a>`);

      return { content, contentBlocks };
    };

    const showContentBlock = (contentBlock, contentBlocks) => {
      document.querySelectorAll('.contentBlock-content').forEach(element => {
        element.style.display = 'none';
        element.classList.remove('visible');
      });
      document.querySelectorAll(`.contentBlock-content[data-contentBlock="${contentBlock}"]`).forEach(element => {
        element.style.display = 'inline';
        setTimeout(() => element.classList.add('visible'), 50);
      });
      localStorage.setItem('activeContentBlock', contentBlock);
    };

    const addSwitchListeners = (contentBlocks) => {
      document.querySelectorAll('.contentBlock-switch').forEach(switchElement => {
        switchElement.addEventListener('click', event => {
          event.preventDefault();
          showContentBlock(switchElement.getAttribute('data-contentBlock'), contentBlocks);
        });
      });
    };

    const workAreas = document.querySelectorAll(scanSelector);
    if (!workAreas.length) {
      console.error('No areas found to scan and modify');
      return;
    }

    const contentBlocks = {};
    workAreas.forEach(workArea => {
      workArea.classList.add('hidden');
      workArea.innerHTML = findAndReplaceText(workArea.innerHTML, pageSpecificReplacements);
      const result = replaceContentBlocks(workArea.innerHTML);
      workArea.innerHTML = result.content;
      Object.assign(contentBlocks, result.contentBlocks);
    });

    addSwitchListeners(contentBlocks);
    const storedContentBlock = localStorage.getItem('activeContentBlock');
    if (storedContentBlock && contentBlocks[storedContentBlock]) {
      showContentBlock(storedContentBlock, contentBlocks);
    } else if (Object.keys(contentBlocks).length > 0) {
      showContentBlock(Object.keys(contentBlocks)[0], contentBlocks);
    }

    workAreas.forEach(workArea => {
      workArea.classList.remove('hidden');
      setTimeout(() => workArea.classList.add('visible'), 50);
    });

    const applyTextReplacements = () => {
      const activeContentBlock = localStorage.getItem('activeContentBlock');
      if (!activeContentBlock) {
        console.warn('No active content block found in memory');
        return;
      }

      const replacements = outsideSelectorReplacements[activeContentBlock];
      if (!replacements) {
        console.warn(`No replacements found for content block: ${activeContentBlock}`);
        return;
      }

      document.querySelectorAll('*:not(script):not(style)').forEach(node => {
        if (node.childNodes.length === 1 && node.childNodes[0].nodeType === 3) {
          node.textContent = findAndReplaceText(node.textContent, replacements);
        }
      });
    };

    applyTextReplacements();
  });
</script>

<!-- Page-Specific Replacements -->
<script>
  window.pageSpecificReplacements = {
    'old text': 'new text',
    'old code': 'new code'
  };
</script>
