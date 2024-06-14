# Konkretnik Content Switcher for Ghost CMS
Ghost injection script for content switching. https://konkretnik.com

## Introduction

This repository provides a Content Switcher script designed specifically for Ghost CMS. The purpose of this script is to allow for easy content switching and text replacement without the need to modify the core Ghost code or themes. This approach ensures better maintenance and flexibility, allowing you to manage content dynamically across different pages.

## Features

- **Dynamic Content Switching:** Easily switch content blocks within specified areas.
- **Text Replacement:** Replace text globally within specified selectors or across the entire site.
- **No Theme Modification Required:** Works independently of the main Ghost themes, ensuring easy updates and maintenance.

## Version

Current version: 0.3.3

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
Author: Bartosz Kulczyk
Website: https://konkretnik.com
License: MIT
Version: 0.3.3
-->

<style> 
  /* Selectors area for content switcher */
  
  body>div.gh-site>main>div>article,
  #gh-head>div>nav
  
  /* End of selectors area */
  
  /* Styling */
  {opacity:0;animation:fadeIn .5s linear forwards}@keyframes fadeIn{0%,50%{opacity:0}100%{opacity:1}}.hidden{display:none}.fade-in{opacity:0;transition:opacity .5s}.fade-in.visible{opacity:1}.contentBlock-content{display:none}.contentBlock-content.visible{display:inline}.hidden{display:none}.active{font-weight:bold}
</style>

<script>
  // Configuration for the content switcher and text replacements
  window.contentSwitcherConfig = {
    // Selector to scan areas for content switching
    scanSelector: 'body>div.gh-site>main>div>article, #gh-head>div>nav',
    // Replacement texts that apply within specified selectors
    selectorReplacementTexts: {
      '<a href="https://konkretnik.com/">ChangeLang</a>':'||en||[[pl]]PL[[/pl]]||/en||||pl||[[en]]EN[[/en]]||/pl||',
      '<a href="https://konkretnik.com/">About</a>':'||en||<a href="https://konkretnik.com/">About me</a>||/en||||pl||<a href="https://konkretnik.com/">O mnie</a>||/pl||',
      '<a href="https://konkretnik.com/konkretnik-content-switcher/">Content Switcher</a>':'||en||<a href="https://konkretnik.com/konkretnik-content-switcher/">Content Switcher</a>||/en||||pl||<a href="https://konkretnik.com/konkretnik-content-switcher/">Podmieniacz treści</a>||/pl||',
      'Get rid with IT overload!':'||en||Get rid with IT overload!||/en||||pl||Ogarnij strukturę IT!||/pl||'
    },
    // Custom replacements outside specified areas
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
    // Combine selector replacements with page-specific replacements
    const pageSpecificReplacements = { ...window.pageSpecificReplacements, ...selectorReplacementTexts };
    
    /**
     * Function to find and replace text in the content
     * @param {string} content - The content to be modified
     * @param {object} replacements - The replacement pairs where key is the text to find and value is the replacement text
     * @returns {string} - The modified content
     */
    const findAndReplaceText = (content, replacements) => {
      for (const [search, replace] of Object.entries(replacements)) {
        // Escape special characters for regex
        const escapedSearch = search.replace(/[.*+?^${}()|[\]\\]/g, '\\$&');
        // Replace text using regex
        content = content.replace(new RegExp(escapedSearch, 'g'), replace);
      }
      return content;
    };

    /**
     * Function to replace content blocks based on custom tags
     * @param {string} content - The content to be modified
     * @returns {object} - An object containing the modified content and a map of content blocks
     */
    const replaceContentBlocks = (content) => {
      // Regex patterns for different custom tags
      const contentBlockPattern = /\|\|(\w+)\|\|([\s\S]*?)\|\|\/\1\|\|/g;
      const switchPattern = /\[\[(\w+)\]\]([\s\S]*?)\[\[\/\1\]\]/g;
      const stylePattern = /%%([^%]+?)\/%%/g;
      const contentBlocks = {};

      content = content
        .replace(contentBlockPattern, (match, contentBlock, text) => {
          contentBlocks[contentBlock] = contentBlocks[contentBlock] || [];
          contentBlocks[contentBlock].push(text);
          return `<span class="contentBlock-content" data-contentBlock="${contentBlock}" style="display: none;">${text}</span>`;
        })
        .replace(switchPattern, (match, contentBlock, text) => `<a href="#" class="contentBlock-switch" data-contentBlock="${contentBlock}">${text}</a>`)
        .replace(stylePattern, (match, style, text) => `<span style="${style}">${text}</span>`);

      return { content, contentBlocks };
    };

    /**
     * Function to show the specified content block
     * @param {string} contentBlock - The identifier of the content block to show
     * @param {object} contentBlocks - The map of all content blocks
     */
    const showContentBlock = (contentBlock, contentBlocks) => {
      // Hide all content blocks
      document.querySelectorAll('.contentBlock-content').forEach(element => {
        element.style.display = 'none';
        element.classList.remove('visible');
      });
      // Show the specified content block
      document.querySelectorAll(`.contentBlock-content[data-contentBlock="${contentBlock}"]`).forEach(element => {
        element.style.display = 'inline';
        setTimeout(() => element.classList.add('visible'), 50);
      });
      // Store the active content block in local storage
      localStorage.setItem('activeContentBlock', contentBlock);
    };

    /**
     * Add event listeners to content block switch elements
     * @param {object} contentBlocks - The map of all content blocks
     */
    const addSwitchListeners = (contentBlocks) => {
      document.querySelectorAll('.contentBlock-switch').forEach(switchElement => {
        switchElement.addEventListener('click', event => {
          event.preventDefault();
          showContentBlock(switchElement.getAttribute('data-contentBlock'), contentBlocks);
        });
      });
    };

    // Find and process the work areas for content switching
    const workAreas = document.querySelectorAll(scanSelector);
    if (!workAreas.length) {
      console.error('No areas found to scan and modify');
      return;
    }

    // Process and replace content blocks in the work areas
    const contentBlocks = {};
    workAreas.forEach(workArea => {
      workArea.classList.add('hidden');
      workArea.innerHTML = findAndReplaceText(workArea.innerHTML, pageSpecificReplacements);
      const result = replaceContentBlocks(workArea.innerHTML);
      workArea.innerHTML = result.content;
      Object.assign(contentBlocks, result.contentBlocks);
    });

    // Add switch listeners to the content block switches
    addSwitchListeners(contentBlocks);
    const storedContentBlock = localStorage.getItem('activeContentBlock');
    if (storedContentBlock && contentBlocks[storedContentBlock]) {
      showContentBlock(storedContentBlock, contentBlocks);
    } else if (Object.keys(contentBlocks).length > 0) {
      showContentBlock(Object.keys(contentBlocks)[0], contentBlocks);
    }

    // Reveal the work areas with a fade-in effect
    workAreas.forEach(workArea => {
      workArea.classList.remove('hidden');
      setTimeout(() => workArea.classList.add('visible'), 50);
    });

    /**
     * Function for text replacement outside specified areas
     */
    const applyTextReplacements = () => {
      const activeContentBlock = localStorage.getItem('activeContentBlock');
      if (!activeContentBlock) {
        console.warn('No active content block found in memory');
        return;
      }

      // Get replacements for the active content block
      const replacements = outsideSelectorReplacements[activeContentBlock];
      if (!replacements) {
        console.warn(`No replacements found for content block: ${activeContentBlock}`);
        return;
      }

      // Replace text throughout the entire website outside specified areas
      document.querySelectorAll('*:not(script):not(style)').forEach(node => {
        if (node.childNodes.length === 1 && node.childNodes[0].nodeType === 3) {
          node.textContent = findAndReplaceText(node.textContent, replacements);
        }
      });
    };

    // Apply text replacements on page load
    applyTextReplacements();
  });
</script>

<!-- Page-Specific Replacements -->
<script>
  window.pageSpecificReplacements = {
    // Page-Specific Replacement Texts (customize as needed)
    'old text': 'new text',
    'old code': 'new code'
  };
</script>
