# Design Verification Page

## Purpose

This document describes a single-page layout composed of common UI components. It is intended to be used alongside a separate design guideline or system prompt. The goal is to generate a page that tests whether the design aesthetic is being understood and applied correctly across a broad set of elements. The page itself is a fictional product or brand landing page — the content is secondary. What matters is that every component gives the designer (or reviewer) something to evaluate.

Think of this page as a **stress test for a design system**. If the aesthetic holds across all of these components, the guideline is working.

---

## Page Structure

The page is a single continuous scroll. It is divided into the following sections, top to bottom. Each section contains one or more components. All sections should feel like they belong to the same page and the same brand.

---

## 1. Navigation Bar

A fixed or sticky top navigation bar.

**Contains:**
- A brand name or wordmark on the leading side
- Four to five navigation links (e.g., Work, About, Services, Journal, Contact)
- A single call-to-action button on the trailing side (e.g., "Get in Touch")

**What this tests:**
- Typographic hierarchy at small sizes
- Spacing and alignment discipline
- Button styling at its most minimal
- How the design handles a horizontal row of equally weighted items

---

## 2. Hero Section

A large, visually dominant section immediately below the nav.

**Contains:**
- A large headline (8–12 words maximum). Example: "We design systems that communicate with clarity."
- A shorter subheadline or supporting sentence beneath it (one to two lines)
- A primary action button and an optional secondary text link
- A large image, illustration, or geometric visual element occupying a significant portion of the section

**Layout note:** The headline and image should have an asymmetric relationship — they should not be centered together like a PowerPoint slide. One should anchor left, the other right, or the image should bleed to an edge.

**What this tests:**
- Display typography — the largest type on the page
- Handling of whitespace at scale
- Image or visual treatment (crop, contrast, color handling)
- Visual weight and compositional balance
- Primary button styling

---

## 3. Metrics / Stats Row

A horizontal row of three to four key statistics or numbers.

**Contains:**
- A large number or value (e.g., "140+", "$2.4M", "99.8%")
- A short label beneath each number (e.g., "Projects Delivered", "Revenue Generated", "Uptime")

**What this tests:**
- Numerical typography and how large figures are styled
- Consistent spacing across repeated elements
- Restraint — this section should feel calm, not loud
- Alignment to the same grid used everywhere else

---

## 4. Two-Column Text and Image Block

A section that pairs a block of body text with a supporting image, side by side.

**Contains:**
- A section heading (e.g., "Our Approach")
- Two to three short paragraphs of body copy (real or plausible placeholder — avoid lorem ipsum if possible)
- An image or visual element beside the text, aligned to the grid

**What this tests:**
- Body text styling: font size, line height, line length, paragraph spacing
- How the design handles longer-form reading
- Image alignment and proportion relative to text
- The relationship between heading and body copy

---

## 5. Card Grid

A grid of three to six cards. These represent items like projects, services, articles, or team members.

**Contains per card:**
- An image or colored placeholder area at the top
- A card title
- A short description (one to two sentences)
- An optional tag, date, or category label
- An optional text link (e.g., "Read more" or "View project")

**Layout note:** Cards should be arranged in a grid (2 or 3 per row). All cards must be the same height or follow a consistent masonry logic.

**What this tests:**
- Component-level consistency and repetition
- How images are cropped and contained
- Small-scale typographic hierarchy (title vs. description vs. metadata)
- Card borders, spacing, and background treatment
- Whether the grid system is actually being used

---

## 6. Featured Quote or Testimonial

A single prominent quotation, centered or offset.

**Contains:**
- A quotation (two to three sentences)
- Attribution: a name and a role or organization
- Optional: a small avatar or photo

**What this tests:**
- How the design handles a single typographic moment — is it elegant or clumsy?
- Quotation mark styling or lack thereof
- Alignment and spacing in a minimal component
- Whether accent color or special treatment is used, and if it fits

---

## 7. Alternating Feature Sections

Two feature blocks stacked vertically. Each contains an image and text, but they alternate sides (image-left/text-right, then text-left/image-right).

**Contains per block:**
- A feature heading
- A short paragraph of supporting text
- A relevant image or graphic

**What this tests:**
- Compositional rhythm and visual pacing
- Whether the alternation feels intentional and structured or arbitrary
- Consistent sizing and spacing between the two blocks
- Image treatment consistency

---

## 8. Form Section

A simple contact or signup form.

**Contains:**
- A heading (e.g., "Start a Conversation" or "Subscribe")
- Two to three text input fields (e.g., Name, Email, Message)
- Input labels (above or as floating/placeholder labels)
- A submit button

**What this tests:**
- Input field styling: borders, padding, focus states
- Label typography and placement
- Button consistency with other buttons on the page
- Form layout and alignment
- Spacing between form fields

---

## 9. Footer

A full-width footer at the bottom of the page.

**Contains:**
- The brand name or wordmark repeated
- Organized link groups (e.g., a column for "Company", a column for "Resources", a column for "Social")
- Three to four links per group
- A copyright line at the bottom
- Optional: social media icon links

**What this tests:**
- Small text styling and legibility
- Multi-column layout at a smaller scale
- Visual separation from the main content (background color shift, rule, or spacing)
- Whether the footer feels like it belongs to the same design system as the rest of the page

---

## 10. Utility and Inline Elements

These elements should appear naturally within the sections above, not as a separate section. They exist to test fine-grained styling.

**Distribute throughout the page:**
- At least one inline text link within body copy
- At least one ordered or unordered list within body text
- A horizontal rule or divider between two sections
- A small badge or tag element (e.g., "New", "Featured", a category label)
- A tooltip or hover state on at least one interactive element
- At least one icon used alongside text (e.g., an arrow icon on a link, a checkmark in a list)

**What this tests:**
- Whether the design system extends to small, incidental elements
- Link styling within paragraphs (color, underline, hover)
- List styling (bullets/numbers, indentation, spacing)
- Divider treatment (thickness, color, spacing)
- Micro-interactions and hover behavior

## 11. Component Architecture

Build components using the shadcn/ui wrapping pattern: Radix primitives for behavior, Tailwind for styling via `cn()`, and `cva` for variant management.

**Key conventions:**

- `forwardRef` on all components -- consumers must be able to attach refs
- `className` pass-through on every component root -- never block consumer overrides
- CSS variables for theming (`--primary`, `--radius`, etc.) -- no runtime CSS-in-JS
- Sensible defaults for all variant props -- components should render well with zero config
- Compose from Radix primitives rather than building interaction logic from scratch

**Wrapping pattern:**

```tsx
const Button = React.forwardRef<
  HTMLButtonElement,
  React.ButtonHTMLAttributes<HTMLButtonElement> & VariantProps<typeof buttonVariants>
>(({ className, variant, size, ...props }, ref) => (
  <button
    className={cn(buttonVariants({ variant, size }), className)}
    ref={ref}
    {...props}
  />
));
```

**What this tests:**
- Prop API consistency -- do all components follow the same `className` + variant pattern?
- Variant coverage -- does `cva` handle size/color/state without one-off conditionals?
- Override flexibility -- can consumers restyle without fighting the component?
- Theme integration -- do components respond to CSS variable changes?

---

## General Evaluation Criteria

When reviewing the generated page, assess the following:

**Consistency** — Do all components feel like they belong to the same design system? Are fonts, colors, spacing, and treatments uniform throughout?

**Grid Integrity** — Is there an underlying alignment structure? Do elements snap to a shared grid, or do they float arbitrarily?

**Typographic Hierarchy** — Can you immediately distinguish headlines from subheads from body text from labels? Is the hierarchy achieved through size, weight, and spacing rather than decoration?

**Whitespace** — Is the page breathing? Does negative space feel intentional and generous, or is everything cramped?

**Color Discipline** — Is color used sparingly and purposefully? Does the palette match the guideline?

**Image Treatment** — Are images handled consistently (crop, aspect ratio, contrast, color treatment)?

**Responsiveness Readiness** — Even as a desktop-first design, do the components feel like they could reflow to a narrower viewport without breaking?

**Restraint** — Is the design solving problems or decorating? Every visual choice should be traceable to a reason.

---

## Usage

Pair this document with a design system prompt (your aesthetic guideline). Provide both to the model with an instruction like:

> "Using the design guideline provided, generate a single-page website that implements the components described in the design verification page. The page should be a cohesive, working example of the design system applied across all listed components."

The output should be a single HTML file (with embedded or linked CSS) that can be opened in a browser for visual review.
