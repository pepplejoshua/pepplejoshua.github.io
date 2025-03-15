---
layout: post
title:  "learning webdev: implementing a 'google docs-like' title bar in remix"
date:   2025-03-11 14:03:00 -0700
categories: programming ts webdev
---

I was working on an inline-editable title for my application and much like my [last entry](https://pepplejoshua.github.io/programming/ts/webdev/2025/01/01/learning-webdev-sessions.html),
I took inspiration from an application I use often: Google Docs. In this case, it is Google Docs's Title. You know the one - where the title sits at the top left of the document,
looking like regular text until you hover over it, then seamlessly transforms into an editable field when clicked.

In summary,

1. it shows the border of the input field when you hover over it.
2. when you click on it, it allows editing the title.
3. the input field grows and shrinks dynamically with the content. When it grows or shrinks, there is no jitter when the width of the field is recomputed.
4. by default, scroll is not enabled (unless you have reached the max-width of the field).
5. between hover and click states, the border shown is exactly the same, so the Ui looks cleaner.

Even with Claude's help, this took longer than I expected. Building the Api and title updating functionality (effectively
the backend code) took no time. Implementing the inline-editable title took me through so many phases of hell. No matter what I tried, nothing worked correctly
for all edge cases. I tried

1. The Character Width Calculation method: computing the width of each character typed, but this would cause the field to jitter. Also not very efficient either.
2. The Manual Hidden Span approach: using a hidden span with the same style to compute the width of the actual field. This would for some reason not work and the field would
have scroll issues or the width of the field would be out of sync for things like `cmd-z` on entered text.
3. The Monospace Font solution: using monospace which was quickly scraped because it would go away from the design I have for the app.

I eventually gave up and just went for a single input field that had a fixed width but no hover border. I hated it, so I worked with Claude to make it look nicer it.
And then I kept fixing small issues with the simple field. I fixed the collapsing problem (where an empty field would look literally empty and entering text would just make it scroll).
Then I tried one more time looking at Google Docs's code using *Inspect Element* in the browser. I noticed it had some strange span whose content updated as the text field was typed into.
The width of the input field also grew exactly with the width of the specific character entered. I showed the code to Claude and asked it to try using this information to write the code.
I had done this previously but it yielded nothing useful.

This time, it actually solved it first time using code I was not familiar with. So here I am, documenting it.
This blog will summarize the `InlineEditor` component solves the problems I set out to solve.


### Properties and Internal State
---

It takes a few properties, mostly to handle how to respond to user actions. The interface is defined as:
```tsx
interface InlineEditorProps {
  value: string;              // Starting value of the input field
  onChange: (newValue: string) => void;  // Handler for value changes
  onSave: () => void;        // Handler for when edit is confirmed
  onCancel: () => void;      // Handler for when edit is cancelled
  maxLength?: number;         // Optional character limit
  className?: string;        // Optional CSS classes
}
```

For internal state, it has 3:
1. Whether it's in edit mode or not.
```tsx
const [isEditing, setIsEditing] = useState(false);
```

2. The value from an edit
```ts
const [editValue, setEditValue] = useState(value);
```

3. The current width of the visible input field
```ts
const [width, setWidth] = useState<number | null>(null);
```

4. A reference to the hidden span for direct DOM access
```ts
const measureRef = useRef<HTMLSpanElement>(null);
```

5. A reference to the visible input field for direct DOM access
```ts
const inputRef = useRef<HTMLInputElement>(null);
```

### Hidden Span, Visible Input and Width Calculation
---

The hidden span always contains the same content and styling as the input field at any given time. This is useful for computing the
width of the visible input field. It looks like this:
```tsx
<span
  ref={measureRef}
  aria-hidden="true"
  className="invisible absolute whitespace-pre"
  style={{
    font: "inherit",
    fontSize: "1.25rem",
    fontWeight: 600,
    padding: "0.5rem",
  }}
>
  {editValue}
</span>
```

`invisible` and `absolute` classes allow the span to be hidden but still rendered (imagine computing the width on something that does not exist in the DOM).
`whitespace-pre` preserves the spaces exactly as they are typed into the input field. It can be directly accessed or referenced through `measureRef`.

The visible input field looks like this:
```tsx
<input
  ref={inputRef}
  type="text"
  value={editValue}
  onChange={handleChange}
  onKeyDown={handleKeyDown}
  onClick={handleClick}
  onBlur={handleBlur}
  maxLength={maxLength}
  readOnly={!isEditing}
  spellCheck={false}
  className={`
    bg-transparent
    text-xl font-semibold text-white
    h-9 px-2 rounded border-2
    focus:outline-none
    ${
      isEditing
        ? "border-gray-300/50"
        : "border-transparent hover:border-gray-500/50 hover:bg-gray-900/50 cursor-pointer"
    }
  `}
  {% raw %}style={{
    width: width ? `${width}px` : "auto",
  }}{% endraw %}
  title={!isEditing ? "Click to edit" : undefined}
/>
```

It is one chunky input. It has spellchecking disabled to remove those annoying spellcheck underlines, allows toggling read-only mode,
has a length limit for character count, accepts events handlers for edits, clicks, pressing esc or enter, clicking off it as well as its
computed width. It can be directly accessed through `inputRef`. The internal event handlers are:
```ts
// handle special case buttons like Escpae and Enter
const handleKeyDown = (e: React.KeyboardEvent<HTMLInputElement>) => {
  if (e.key === "Enter") {
    e.preventDefault();
    setIsEditing(false);
    onSave();
  } else if (e.key === "Escape") {
    e.preventDefault();
    setIsEditing(false);
    onCancel();
    inputRef.current?.blur();
  }
};

// handle changes in the input field
const handleChange = (e: React.ChangeEvent<HTMLInputElement>) => {
  if (isEditing) {
    setEditValue(e.target.value);
    onChange(e.target.value);
  }
};

// handle a click on the input field
const handleClick = () => {
  if (!isEditing) {
    setIsEditing(true);
  }
};

// handle blur / cancel of the edit operation
const handleBlur = () => {
  setIsEditing(false);
  onCancel();
};
```

In order to efficiently detect changes (changes in width and value) for the hidden span, I use a `ResizeObserver` and a `MutationObserver`.

1. `ResizeObserver`: A browser API that watches for changes in an element's size
2. `MutationObserver`: A browser API that watches for changes in DOM elements

They are set up in a `useEffect()` that runs only when the component first mounts. That code looks like this:
```ts
useEffect(() => {
  if (!measureRef.current) return; // if the node for the hidden span does not exist, return

  // the function that computes the width of the hidden span
  // and updates the width of the visible field
  const updateWidth = () => {
    const spanWidth = measureRef.current?.offsetWidth || 0;
    setWidth(spanWidth + 8); // 8 is for padding
  };

  // ResizeObserver watches for size changes
  const resizeObserver = new ResizeObserver(updateWidth);
  // It will observe the hidden span and call updateWidth when its size changes
  resizeObserver.observe(measureRef.current);

  // MutationObserver watches for content changes
  const mutationObserver = new MutationObserver(updateWidth);
  // It will observe the hidden span and call updateWidth when its content changes
  mutationObserver.observe(measureRef.current, {
    // watch for text content changes
    characterData: true,
    // watch all descendants, including the internal text content node. prevents visual janks
    subtree: true,
  });

  // when the InlineEditor component is unmounted, we want to clean up our observers with
  // this function
  return () => {
    resizeObserver.disconnect();
    mutationObserver.disconnect();
  };
}, []);
```

This is simpler than most of all the other ways I tried, which is usually how great solutions to problems tend to turn out.
`ResizeObserver` and `MutationObserver` are both Web APIs with great browser support, making them perfect for the job.

So when a user edits an input field, the `editValue` variable changes, which changes the hidden span. This hidden span is
watched by 2 observers for changes in size and content. If either observer notices a change, it calls a function that recomputes
the width of the hidden span and smoothly updates the width of the visible input.

What I still can't understand is why it works out smoothly in the Ui without any jank. When I ask Claude why this works out better
compared to other solutions, it says (summarized as):

1. The Character Width Calculation method tries to measure each character's width and sum them up, without accounting for font
kerning. This leads to incorrect calculations and jittery updates, while being computationally expensive. This felt directionally correct
but I was misled by observing how the width changed by different integer values based on the character entered since some
characters are wider than others in non-monospace fonts (for example, w vs i).

2. The Manual Hidden Span approach attempts to sync an input with a hidden measuring element. I failed to cover the edge
cases for all updates (like undo/redo) and it suffered from render timing issues, just like the CWC method and sometimes the width of
the input field would be wildly incorrect. It took too much effort with no obvious profit.

3. The Monospace Font solution forced me to use a font I had no interest in using. Never a solution.

The Observer Pattern, which ended up working perfectly, succeeds by using browser-native APIs to automatically detect and respond to
content and size changes, letting the browser handle all timing and measurement complexities. It covers all the edge cases I didn't
from the MHS approach (typing, pasting, undo/redo) and simplifies the code by eliminating the need to manually compute widths or determine
when some Ui component has changed size and then try to respond in time to render smoothly. I wish there was a browser / React expert I could
ask about these things.

The final `InlineEditor` is used like so:
```tsx
<InlineEditor
  value={initialTitleValue}
  onChange={onTitleChangeHandler}
  onSave={saveTitleHandler}
  onCancel={() => setTitle(originalTitle)}
  maxLength={75}
/>
```

### Takeaways
1. The simplest solution isn't always immediately obvious, especially in the absence of experience.
2. The browser is very powerful, and its Apis can provide simple solutions to application layer problems.
3. Understanding how the browser and the framework I am using works is crucial to building the best Ui and User experience
possible.
4. I have a lot to learn.

I enjoyed solving this because it allowed me to see a different side of frontend development. I have been using all kinds of
browser Apis this whole time and taking them for granted. I feel very differently about web development now. It is still frustrating,
but I am making progress.

*jp*
