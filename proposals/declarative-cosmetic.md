# DeclarativeCosmeticRules API proposal

## Background

### Cosmetic rules in content blockers

Cosmetic rules can be divided into three groups: element hiding rules, CSS rules, and scriptlets.

- Element hiding rules can be used to hide various elements on web pages, such as advertisements, pop-ups, banners, and other unwanted content. By defining the CSS selectors of these elements, users can hide them from view for a more pleasant browsing experience.

- CSS rules can be used to add different styles to DOM elements.

- Scriptlets can be used to modify JS behavior, abort retrieval of some props, speed up timers, abort inline scripts, remove DOM element attributes or classes, etc.

### How cosmetic rules are applied in MV2 and MV3?

#### MV2

The extension needs to inject scripts and styles as early as possible for a smoother user experience (e.g. blinking DOM elements). It also needs to patch scripts before websites can copy DOM API methods. This forced the extension to use a rather sophisticated way of injecting scripts and styles based on events thrown by the webRequest and webNavigation APIs. In short, at webRequest.onHeadersReceived, when the first information of the request is received, the extension asks the engine for the rules related to the current request and prepares styles and scripts to inject. As the engine is already running, this information can be obtained very quickly. At webRequest.onResponseStarted, the extension tries to inject the scripts received in the previous step. This event is not reliable, so at webNavigation.onCommitted the extension will inject scripts again if they weren't injected before. Along with the scripts, the extension will also inject CSS styles.

#### MV3

Extensions built on top of MV3 can use the same sophisticated way to inject styles and scripts, but they lack the guarantee that the engine will start. This is because event-driven background pages or service worker pages can, as we know, die. So it takes time for the search engine to start, it injects with some delay, and the user gets a bad experience.

#### How many cosmetic rules are there?

Element hiding rules are one of the most popular rule types - for example, AdGuard's basic filter contains 98500 rules, 24800 of which are element hiding rules.

CSS rules and scriptlets are less common. However, they are still very popular among filter developers, especially in some difficult cases.
Scriptlet rules make up 3000 rules and cosmetic CSS rules make up 1500 rules.

## Goal

We need to be able to apply cosmetic rules before the page loads, without having to use content scripts and complicated logic. We also do not want to lose any current features.

## API Proposal

### Declarative element hiding rules

```ts
type Rule = {
    action: RuleAction,
    condition: RuleCondition,
};

type RuleAction = {
    type: RuleActionType,
    selector: string,
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
 * "hide" - hides the element with the selector
 */
type RuleActionType = 'hide';

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
```

// TODO add examples with allowlist rules

### Declarative css rules
// TODO

### Declarative scriptlets rules
// TODO

### API to manage rules dynamically
// TODO
