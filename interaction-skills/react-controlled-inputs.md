# React-controlled inputs

Setting an input's value with `type_text` (CDP `Input.insertText`) or even `el.value = ...` updates what you *see*, but a React-controlled input keeps its own state. The framework's `onChange` never fires, so React's state still holds the OLD value. The visible text and the green "valid" check lie: when you submit, the form posts the stale value and your change silently vanishes.

This bites hardest on legacy admin forms (Jira `/secure/admin/*`, Confluence, Salesforce setup) where a coordinate-click + `type_text` + Save *looks* like it worked, the page navigates away, and a reload shows the field unchanged.

## Symptom

- You typed into the field, screenshot shows the new value, validation passed.
- You clicked Save. No error. Maybe a redirect.
- Reopen the form → the field reverted to its original value.

The change was never in React state, so Save submitted the old value.

## Fix: native setter + dispatched events

Call the prototype's *native* value setter (React patches the instance setter to detect direct assignment, but reads through the prototype). Then dispatch real `input` + `change` events so React's synthetic system updates state.

```python
js(r'''(function(){
  const el = [...document.querySelectorAll('input,textarea')]
    .find(e => /SOME UNIQUE CURRENT VALUE/.test(e.value||''));   // target your field
  if(!el) return 'NOINPUT';
  const proto = el.tagName==='TEXTAREA'
    ? window.HTMLTextAreaElement.prototype
    : window.HTMLInputElement.prototype;
  Object.getOwnPropertyDescriptor(proto,'value').set.call(el, 'NEW VALUE');
  el.dispatchEvent(new Event('input',  {bubbles:true}));
  el.dispatchEvent(new Event('change', {bubbles:true}));
  return el.value;
})()''')
```

Then submit by calling the button element's `.click()` directly (not a coordinate click — see below):

```python
js(r'''(function(){
  const b=[...document.querySelectorAll('button')].find(e=>/^save$/i.test((e.textContent||'').trim()));
  if(!b) return 'NOSAVE';
  if(b.disabled) return 'DISABLED';
  b.click();
  return 'clicked';
})()''')
```

## Why also click via `.click()`, not coordinates

Two independent failure modes stack on these forms:

1. **Value not in state** (fixed by the native setter above).
2. **Save click misses.** Screenshots are downscaled — reading a pixel off the image puts the click in the wrong place, and the form silently doesn't submit. `element.click()` always hits the real button. Prefer it for the *submit* of a fiddly form even though coordinate clicks are the harness default elsewhere.

## Verify persistence — don't trust the redirect

A post-save redirect is not proof. Reopen the edit form (or re-fetch via API) and read the value back. The summary/list view is often cached and will show the new value even when the save failed, or the old value even when it succeeded — read the authoritative source.

```python
# reopen the edit form, then:
js(r'''(function(){const e=[...document.querySelectorAll('input,textarea')]
  .find(e=>/objectType|SOME ANCHOR/.test(e.value||''));return e?e.value:'NF';})()''')
```

## When you need this

- The field is React/Vue-controlled (most modern SPA forms; all of Jira/Confluence Cloud admin).
- `type_text` shows the value but Save doesn't stick.
- A plain `<form>` with uncontrolled inputs does NOT need this — `type_text` + real submit works there. Reach for the native setter only after a save silently reverts.
