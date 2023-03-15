# DeclarativeCosmeticRules API proposal

## Background

### Cosmetic rules in content blockers

Cosmetic rules can be divided into three groups: element hiding rules, CSS rules, and scriptlets.

- Element hiding rules can be used to hide various elements on web pages, such as advertisements, pop-ups, banners, and other unwanted content. By defining the CSS selectors of these elements, users can hide them from view for a more pleasant browsing experience. Technically, element hiding rules inject a CSS `display:none` style into the page for a given element.

- CSS rules can be used to add different styles to DOM elements. Technically, CSS rules inject a custom CSS style into the page. There are some restrictions on what styles can be injected, e.g. you cannot use a style that loads additional resources. For example, `url()`, etc.

- Scriptlets can be used to modify JS behavior, abort retrieval of some props, speed up timers, abort inline scripts, remove DOM element attributes or classes, etc. Technically, scriptlets change the behaviour of the page by executing small named JS functions that come with the extension. Example: `abort-property-read(propName)`.

### Main issues

* Content blocking extensions require wide permissions, mostly to apply cosmetic rules. This is not secure and may scare some people that the extension may be watching them.

* Timing. Content blocking extensions would like to apply cosmetic rules as quickly as possible, that is, before the page loads and page scripts start executing. With the current approach, there is a slight delay. It would be ideal if the new API applied the rules after merging the CSSDOM and DOM trees built and before the layout step.

### How cosmetic rules are applied in MV2 and MV3?

#### MV2

The extension needs to inject scripts and styles as early as possible for a smoother user experience (e.g. blinking DOM elements). It also needs to patch scripts before websites can copy DOM API methods. This forced the extension to use a rather sophisticated way of injecting scripts and styles based on events thrown by the `webRequest` and `webNavigation` APIs. In short, at `webRequest.onHeadersReceived,` when the first information of the request is received, the extension asks the engine for the rules related to the current request and prepares styles and scripts to inject. As the engine is already running, this information can be obtained very quickly. At `webRequest.onResponseStarted`, the extension tries to inject the scripts received in the previous step using `tabs.executeScript`. This event is not reliable, so at `webNavigation.onCommitted` the extension will inject scripts again if they weren't injected before. Along with the scripts, the extension will also inject CSS styles using `tabs.insertCSS`.

So to inject cosmetic rules we have to ask for the next permissions:
* `tabs` - `tabs.insertCSS` to insert styles and `tabs.executeScript` to inject scripts
* `webRequest` - to listen for events
* `webNavigation` - to listen for events
* `<all_urls>` - because we need to inject scripts and styles into all pages
And these permissions are pretty powerful.

#### MV3

Extensions built on top of MV3 injects scripts using `scripting` api and content script for styles. To inject scripts extension subscribes to the `webNavigation.onCommitted` event and injects scripts when this event fires. To inject styles extension uses content script. The content script is injected into every page and requests for the styles from the background page via messaging.

So to inject cosmetic rules we have to ask for the next permissions:
* `scripting` - `scripting.executeScript` to inject scripts and scriptlets
* `webNavigation` - to listen for events and inject scripts in time
* `<all_urls>` - because we need to inject scripts and styles into all pages
* `content_script` - not a permission, but a way to inject styles into the page

#### Why not use a content script to inject the cosmetic rules?

In order to insert styles and scripts selectively, we need to launch the engine to search for the rules suitable for this website only. Launching the engine takes some time, if the engine is used in the content script it would be launched for each website separately. This would lead to significant performance degradation due to large script compilation containing a lot of rules.
Alternatively, the engine could be launched in the background page or service worker, but this would still require time for messaging between the background page and the content script.

#### How many cosmetic rules are there?

Element hiding rules are one of the most popular rule types - for example, AdGuard's Base filter contains 98500 rules, 24800 of which are element hiding rules.

CSS rules and scriptlets are less common. However, they are still very popular among filter developers, especially in some difficult cases.
Scriptlet rules make up 3000 rules and cosmetic CSS rules make up 1500 rules in the AdGuard Base filter.

## Goal

### MV3

One of the goals of MV3 is to make extensions have fewer permissions by default, and to make maximum permissions optional.

### Proposal goal

The goal of this proposal is to make cosmetic rules declarative. This will allow us to remove the `tabs` and `webRequest` permissions from the extension manifest. This will also allow us to remove the `<all_urls>` permission from the extension manifest. Finally, it would allow us not to inject content script into every page.

To avoid reinventing the wheel, we took the Declarative Net Request API as an example, and tried to build logic on its likeness to take advantages of pre-built Declarative CSS rules.

And as a DNR API we need the ability to dynamically change these rules (https://github.com/w3c/webextensions/issues/162) - for CSS rules it's doubly important.

## Several examples of how it can be used

### Declarative element hiding rules

See - https://adguard.com/kb/general/ad-filtering/create-own-filters/#cosmetic-elemhide-rules

```ts
/**
 * "hide" - hides the element with the selector
 */
type RuleActionType = 'hide' | 'allow' | 'css' | 'revert_css' | 'scriptlet';

type Rule = {
    action: RuleAction,
    condition?: RuleCondition,

    /**
     * A list of CSS rules to apply to the element.
     *
     * {@link https://developer.mozilla.org/en-US/docs/Web/API/CSSStyleDeclaration}
     */
    css?: CSSStyleDeclaration

    /**
     * Information about the scriptlet to execute the JS rule
     */
    scriptlet?: ScriptletInfo
};

type RuleAction = {
    type: RuleActionType,
    selector?: string,
};

type ScriptletInfo = {
    name: string,
    args: string[],
};

type RuleCondition = {
    /**
     * List of domains where the action should be applied.
     * If this field is omitted, the rule will be applied to all domains.
     */
    domains?: string[],

    /**
     * List of domains where the action should not be applied.
     */
    excludedDomains?: string[],
};

/**
 * Generic hiding rule e.g. - "##selector"
 */
const genericHidingRule: Rule = {
    action: {
        type: 'hide',
        selector: 'selector',
    },
    // No condition means the rule is applied to all domains.
};

/**
 * Generic hiding rule with exclusion e.g. - "~foo.com##selector"
 */
const genericHidingRuleWithException: Rule = {
    action: {
        type: 'hide',
        selector: 'selector',
    },
    condition: {
        // No domains means rule applies to all domains except those listed in excludedDomains.
        excludedDomains: ['foo.com'],
    },
};

/**
 * Specific hiding rule e.g. - "foo.com##selector".
 */
const specificHidingRule: Rule = {
    action: {
        type: 'hide',
        selector: 'selector',
    },
    condition: {
        domains: ['foo.com'], // This rule would apply to foo.com and all its subdomains.
    },
};

/**
 * Specific hiding rule with exclusion e.g. - "foo.com,~sub.foo.com##selector".
 */
const specificHidingRuleWithException: Rule = {
    action: {
        type: 'hide',
        selector: 'selector',
    },
    condition: {
        domains: ['foo.com'], // This rule would apply to foo.com and all its subdomains.
        excludedDomains: ['sub.foo.com'], // except sub.foo.com
    },
};

/**
 * Allowlist exclusion e.g. - "@@||example.com^$elemhide" - disables all cosmetic rules.
 */
const elemHideRule: Rule = {
    action: {
        type: 'allow',
    },
    priority: 1,
    condition: {
        domains: ['example.com'], // This rule would apply to example.com and all its subdomains
    },
};

/**
 * "###banner" - hides '#banner' on all sites.
 */
const genericHideRule: Rule = {
    action: {
        type: 'hide',
        selector: '#banner',
    },
};
```

### Declarative css rules

See - https://adguard.com/kb/general/ad-filtering/create-own-filters/#cosmetic-css-rules

```ts
/**
 * #$#.textad { visibility: hidden; } - hides '.textad' on all sites via CSS,
 * but not removing from the DOM.
 */

const hideElementRule: Rule = {
    action: {
        type: 'css',
        selector: '.textad',
    },
    css: {
        visibility: 'hidden',
    }
};

/**
 * If you want to disable it for example.com, you can create an exception rule:
 * example.com#@$#.textad { visibility: hidden; }
 */

const excludeHideElementRule: Rule = {
    action: {
        type: 'revert_css',
        selector: '.textad',
    },
    css: {
        visibility: 'hidden',
    }
};

```

### Declarative scriptlets rules

See - https://adguard.com/kb/general/ad-filtering/create-own-filters/#scriptlets

```ts
/**
 * example.org#%#//scriptlet("abort-on-property-read", "alert") - do not allow usage of window.alert on the example.org site.
 */

const hideElementRule: Rule = {
    action: {
        type: 'scriptlet',
    },
    scriptlet: {
        name: 'abort-on-property-read',
        args: ['alert']
    }
};
```

### API to manage rules dynamically
// TODO
