# TestingTutorial

## Introduction
Let's say we have a beautiful pomodoro timer web app like so:

![timer_published_site](source/images/timerPublishedPage.png)

The underlying HTML DOM tree and web components are as follows:

![timer_HTML](source/images/timerHTMLExpose.png)

![timer_DOM_tree](source/images/timerDOMTree.png)


## Cypress & Jest Tests

### Quick Commands
- `npm run jest_test` - run Jest tests
- `npm run jest_coverage` - display Jest coverage
- `npm run cypress_test` - run Cypress tests
- `npm run cypress_start` - open Cypress UI
- `npm i` - install dependencies

<br/>

## Part 1: Writing Cypress Tests
### Opening the Page
Make sure that Cypress finds the correct `index.html` path and can visit the site hosted by LiveServer.
The below assumes that we have a file breakdown like:

```
source/
   index.html
README.md
```

```javascript
describe("Open Page", () => {
    it("Opens index.html", () => {
        cy.visit('./source/index.html')
    });
});
``` 
### Referencing Components & Elements
Finding a component that is a shadow DOM (like `pomo-timer`):
In lab 6, we created a `RecipeCard` custom component. This `pomo-timer` component is
no different and can be defined similarly.
```js
  class pomoTimer extends HTMLElement {
    constructor() {}
  } 
  customElements.define('pomo-timer', pomoTimer);
```
To check that the pomo timer exists in the window, we can test:
```js
describe('Find Timer Element with JS', () => {
  it('Get element (\'Timer\')', () => {
      cy.window().then((win) => {
          expect(win.pomoTimer).to.exist;
      });
  });
});
```

Finding an element by type (e.g. 'p' is paragraph class):
  - **Note**: if this element belongs to a shadow DOM, make sure to include `{includeShadowDom: true}`
```js
describe('Find Elements', { includeShadowDom: true }, () => {
    it('Get element X', () => {
        cy.get('p');
    });
});
```

Finding an element by ID (use a *unique* ID like `timer-button` not `button`):
  - **Note**: if this element belongs to a shadow DOM, make sure to include `{includeShadowDom: true}`
```js
describe('Find Elements', { includeShadowDom: true }, () => {
    it('Get element X', () => {
        cy.get('#id-name');   // cy.get('#timer-button');
    });
});
```

Finding an element by class:
  - **Note**: if this element belongs to a shadow DOM, make sure to include `{includeShadowDom: true}`
```js
describe('Find Elements', { includeShadowDom: true }, () => {
    it('Get element X', () => {
        cy.get('.class-name');
    });
});
```

### Calling Functions
If working with a shadow DOM component, use `cy.window` and `win.component.function()`:
  - **Note**: if this element belongs to a shadow DOM, make sure to include `{includeShadowDom: true}`
Assume that we have a function `setTimer()` in our pomo timer custom component (prototype defined below).
To call that function, we pass in certain values. You can also add a subsequent check to see if the
window changed in an expected fashion.

```js
// function setTimer(min, mode) {}
it('Call setTimer() to set work mode', () => {
  cy.window().then((win) => {
      win.pomoTimer.setTimer(12, 'work');
  });
});
```

If NOT working with shadow DOMs (e.g. local storage file `storage.js`), use `export` in the component JS file and `import` in the Cypress test file:
```js
// storage.js
export function helloWorld(hi) { return hi + " " + "World"; }
```
```js
import { helloWorld } from '../../source/storage'
```
```js
cy
    .wrap({ hw: helloWorld })
    .invoke('hw', 'Hello')
        .should('be.eq', 'Hello World')  // match
```

### Checking Variables
If working with shadow DOM (e.g. `pomo-timer`) to check a `this.variable`:
Assume that we have a global variable `totalSeconds` and we want to confirm its value.
```js
it('Check that total seconds is accurate', () => {
  cy.window().then((win) => {
    expect(win.pomoTimer.totalSeconds).to.eq(num * 60);
  });
});
```

If NOT working with shadow DOMs (e.g. `storage.js`):
```js
// storage.js
let my_num = 0;
export function setNum(num) { my_num = num; }
export function getNum() { return my_num; }
```

```js
import { setNum, getNum } from '../../source/storage'
```

```js
cy
    .wrap({ sn: setNum })
    .invoke('sn', 10)         // set
    .wrap({ gn: getNum })
    .invoke('gn')
        .should('be.eq', 10)  // get
```

### Testing Events
**Note**: all of the following should be encapsulated with a test `it()` and `describe()` <br/>
Basic example of adding and removing an event listener.
```js
it('Basic Event Listener', () => {
  const eventPromise = new Cypress.Promise((resolve) => {
    cy.get('component-id').then(($el) => {
      const eventFunction = (e) => {
        $el[0].removeEventListener('event-name', eventFunction);
        resolve();
      };
      $el[0].addEventListener('event-name', eventFunction);
    });
  });
  cy.wrap(eventPromise);
});
```
Example of clicking button, waiting, and setting a function after event listened.
```js
it('Set Timer for Long Break #1 after listening for Work #4 end', () => {
  cy.get('#timer-button').click();          // do a button click then 
  cy.wait(15000);                           // wait for 2m 
  const eventPromise = new Cypress.Promise((resolve) => {
    cy.get('#pomo-timer').then(($el) => {
      const onFinish = (e) => {
        $el[0].removeEventListener('timerFinish', onFinish);
        $el[0].setTimer(1, 'long break');   // set the timer
        resolve();
      };
      $el[0].addEventListener('timerFinish', onFinish);
    });
  });
  cy.wrap(eventPromise);
});
```

### Comparing Text/Labels
Check text/labels by content. In this case, we want to see that the element with the ID
`timer-mode` has the text `'WORK'`.
```js
it('Mode should be \'Work\'', () => {
  cy.get('#timer-mode').then(($el) => {
    expect($el).to.contain('WORK');
  });
});
```

Check text/labels by class changes
```js
it('Mode should be \'Work\'', () => {
  cy.get('#timer-mode').then(($el) => {
    expect($el).to.have.attr('class', 'work');
  });
});
```

Wait to do an action until text matches
**Note**: you need to add a timeout to wait; for ex 30000 = fast 2m
```js
it('When timer text is 00:00 ...', () => {
  cy.get('#timer-text').contains('00:00', { timeout: 30500 }).then(($el) => {
    // do action
  });
});
```

### Interacting with Buttons
To "click" a button:
```js
it('Button toggles when Start clicked', () => {
  cy.get('#timer-button').click();
  cy.get('#timer-button').then(($el) => {
    expect($el).to.have.attr('class', 'reset');
  });
});
```

<br/>

## Part 2: Writing Jest Tests
TBD
<br/>

## Part 3: Running Jest Tests
TBD
<br/>

## Part 4: Running Cypress Tests
### Locally
- Open VSCode in **source** folder
- Launch LiveServer with `index.html` file
- `npm i`
- `npm run cypress_start`
- **Potential Errors**:
  - If your terminal cannot find `npm`, check that you have it installed with `npm -v`
  - If your terminal cannot find `cypress_start`, check that you have all the `.json` files
  - If Cypress isn't showing the HTML timer but folders, re-open VSCode in the **source** folder (you have the wrong root)
