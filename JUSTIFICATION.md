# Atomic Design Justification

Rules used: **Atom** = StatelessWidget, no logic/state beyond rendering. **Molecule** = groups atoms into one functional unit, may hold local UI state, no business logic. **Organism** = complex section, may hold local logic, must not own core data. **Template** = layout slots as `Widget` parameters, never imports a data model. **Page** = only place that holds the product list and wires data down.

## Atoms (`lib/ui/atoms/`)

**AppHeading** – Atom. It is a StatelessWidget that renders one styled `Text` from a string it is given. It has no logic and no state, so it meets the atom rule. Reused for "Search Products", "Catalog" and "Add New Product".

**AppPrimaryButton** – Atom. It renders an `ElevatedButton` purely from props (label, callback, width, padding, text style) and never decides what a press does. No logic and no state, so it is an atom. Reused for "Add to Cart" and "Submit Product".

**CatalogAppBar** – Atom. Its title and colour are fixed and never depend on app state, so a StatelessWidget is enough. It is a single indivisible UI element with no logic.

**CategoryDropdownField** – Atom. It receives the current value, the options and an `onChanged` callback, and only reports changes upward. The selected value lives in the form organism, so the atom has no state or logic.

**CategoryLabel** – Atom. One styled `Text` showing a category string. No logic, no state.

**DeleteIconButton** – Atom. A single `IconButton` that only fires `onPressed`. What deletion means is decided by the caller, so it holds no logic.

**LabeledTextFormField** – Atom. A generic `TextFormField` whose controller, label, validator, keyboard type and line count all come from outside. It has no opinion on what is valid, so it holds no logic. Reused for Name, Price and Description.

**PriceLabel** – Atom. Renders "PHP xx.xx" from a `double`. Formatting for display is part of rendering, not business logic.

**ProductIconAvatar** – Atom. A 56x56 box around one `IconData`. Pure presentation with no state.

**ProductNameLabel** – Atom. One styled `Text` for the product name. No logic, no state.

**SearchTextField** – Atom. A plain `TextField` that forwards each keystroke through `onChanged` and owns no state. It is deliberately uncontrolled, matching the original behaviour where the visible text does not clear when the query is reset.

## Molecules (`lib/ui/molecules/`)

**CatalogSearchBar** – Molecule. It combines `AppHeading` and `SearchTextField` into the single "search" unit the user sees. It groups atoms into one functional unit and has no business logic. It needs no local state either, which a molecule is allowed but not required to have.

**ProductActions** – Molecule. It groups `AppPrimaryButton` ("Add to Cart") and `DeleteIconButton` into one actions unit. It combines two atoms and holds no logic, only forwarding two callbacks.

**ProductInfo** – Molecule. It groups `ProductNameLabel`, `CategoryLabel` and `PriceLabel` into the "product info" block. It receives a `Product` as a plain parameter, which is allowed. It does not hold or own any list.

## Organisms (`lib/ui/organisms/`)

**ProductCard** – Organism. It is a complete card built from the `ProductIconAvatar` atom and the `ProductInfo` and `ProductActions` molecules, so it is a complex section made of lower levels. It does not own the product list and reports user intent only through callbacks.

**ProductCatalogList** – Organism. It combines the "Catalog" heading with a list of `ProductCard`s and filters them by `searchQuery`. Filtering is local presentation logic, which organisms may have. It never mutates or owns the list, which the Page supplies.

**AddProductForm** – Organism. It is the whole "Add New Product" section. It owns only form-level state: the `GlobalKey<FormState>`, the text controllers and the selected category. It also runs validation (name required; price required, numeric and greater than zero). That is local logic, which organisms may have, and it does not touch core data. On a valid submit it reports raw values upward through `onSubmit` and resets its own fields.

## Template (`lib/ui/templates/`)

**CatalogPageTemplate** – Template. It arranges an app bar plus three `Widget` slots (search, catalog, form) in a scrollable column. It only takes `Widget` parameters and never imports `Product`, which is the template rule. It knows layout and nothing about data.

## Page (`lib/ui/pages/`)

**CatalogPage** – Page. It is the only widget that holds `_products`, `_searchQuery` and `_nextId`. It wires data and callbacks down into the template's slots. It owns the data-changing steps: adding a product, deleting a product, and the confirmation snackbars.

## Where the submit logic ended up

| Piece | Location | Reason |
|---|---|---|
| Field validation | `AddProductForm` (organism) | Local UI logic about the form's own fields, not core data |
| Form controllers, selected category, reset after submit | `AddProductForm` | Form-level state that belongs to the form itself |
| Building the `Product`, adding it to the list, clearing the search | `CatalogPage` | Changes the core product list, which only the Page may own |
| "added to catalog!" snackbar | `CatalogPage` | It is feedback about the catalog change, so it sits with the code that made the change |

## Parts I was unsure how to classify

- **CatalogAppBar** could be argued as an organism, since it is a full `AppBar`. I kept it an atom because it is a single fixed element with no children from lower levels and no state.
- **ProductCatalogList filtering** is borderline. Filtering derives a view of the data and does not change it, so I treated it as allowed local presentation logic. Moving it into the Page would also be defensible.
- **CatalogSearchBar** holds no state, so it is a molecule only by the "groups atoms into one unit" rule, not because of local state.
- **Passing `Product` into `ProductInfo` and `ProductCard`** is fine, because only Templates are barred from importing the model, and these two receive a single item rather than owning the list.
- **Dropdown after submit**: `CategoryDropdownField` uses `initialValue`, which Flutter reads only once. The original screen has the same quirk, so after a submit the stored category resets to "Electronics" but the dropdown may still show the previous choice. I kept this unchanged so the refactor behaves identically to the original.
