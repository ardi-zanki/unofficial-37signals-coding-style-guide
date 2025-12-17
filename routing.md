# Routing Patterns

> Everything is CRUD - resource-based routing over custom actions.

---

## The CRUD Principle

Model every action as a resource with standard CRUD operations:

```ruby
# Bad - custom actions
resources :cards do
  post :close
  post :reopen
  post :assign
  post :archive
end

# Good - resources for everything
resources :cards do
  resource :closure, only: [:create, :destroy]
  resource :assignment, only: [:create, :destroy]
  resource :archival, only: [:create]
end
```

**Why**: Standard REST verbs map cleanly to controller actions. No guessing what HTTP method to use.

## Noun-Based Resources

Turn verbs into nouns:

| Action | Resource |
|--------|----------|
| Close a card | `card.closure` |
| Watch a board | `board.watching` |
| Pin an item | `item.pin` |
| Publish a board | `board.publication` |
| Assign a user | `card.assignment` |

## Nested Resources

Scope resources to their parent:

```ruby
resources :boards do
  resources :cards, only: [:index, :new, :create]
  resource :publication, only: [:create, :destroy]
  resource :entropy, only: [:show, :update]
end

resources :cards do
  resources :comments, only: [:create]
  resource :closure, only: [:create, :destroy]
  resource :goldness, only: [:create, :destroy]
end
```

## Shallow Nesting

Use `shallow: true` to avoid deep nesting:

```ruby
resources :boards, shallow: true do
  resources :cards
end

# Generates:
# /boards/:board_id/cards      (index, new, create)
# /cards/:id                   (show, edit, update, destroy)
```

## Singular Resources

Use `resource` (singular) for one-per-parent resources:

```ruby
resources :cards do
  resource :closure      # A card has one closure state
  resource :watching     # A user's watch status on a card
end
```

## Module Scoping

Group related controllers:

```ruby
namespace :cards do
  resources :drops       # Cards::DropsController
  resources :moves       # Cards::MovesController
end

scope module: :boards do
  resource :publication  # Boards::PublicationsController
  resource :entropy      # Boards::EntropiesController
end
```

## Path-Based Multi-Tenancy

Account ID in URL prefix:

```ruby
# Middleware extracts /:account_id and sets Current.account
# Routes don't need to reference it explicitly

scope "/:account_id" do
  resources :boards
  resources :cards
end
```

## API Versioning (When Needed)

Same controllers, different formats:

```ruby
# No separate API namespace - just respond_to
class CardsController < ApplicationController
  def show
    respond_to do |format|
      format.html
      format.json { render json: @card }
    end
  end
end
```

## Controller Mapping

Keep controllers aligned with resources:

```
app/controllers/
├── cards_controller.rb
├── cards/
│   ├── closures_controller.rb
│   ├── assignments_controller.rb
│   └── drops_controller.rb
├── boards_controller.rb
└── boards/
    ├── publications_controller.rb
    └── entropies_controller.rb
```

## Key Principles

1. **Every action is CRUD** - Create, read, update, or destroy something
2. **Verbs become nouns** - "close" becomes "closure" resource
3. **Shallow nesting** - Avoid URLs like `/a/1/b/2/c/3/d/4`
4. **Singular when appropriate** - `resource` for one-per-parent
5. **Namespace for grouping** - Related controllers together
