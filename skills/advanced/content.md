# Content: Blog Posts & Events

Requires shop modules `blogging: true` and/or `ticketing: true`.

## Blog Posts

### List Posts
```graphql
query Posts($shopId: String!) {
  getPosts(shopId: $shopId) {
    id title content images shopId createdAt
  }
}
```

### Get Post
```graphql
query Post($postId: String!) {
  getPost(postId: $postId) {
    id title content images shopId createdAt
  }
}
```

### Create Post
```graphql
mutation CreatePost($title: String!, $images: [String!]!, $hook: String!, $content: String!, $status: String!, $shopId: String!) {
  createPost(title: $title, images: $images, hook: $hook, content: $content, status: $status, shopId: $shopId)
}
```
`status`: `"published"` or `"draft"`. `hook` is a short excerpt/preview text.

### Update Post
```graphql
mutation UpdatePost($postId: String!, $shopId: String!, $title: String!, $images: [String!], $hook: String!, $content: String!, $status: String!) {
  updatePost(postId: $postId, shopId: $shopId, title: $title, images: $images, hook: $hook, content: $content, status: $status)
}
```

---

## Events

### List Active Events
```graphql
query Events($shopId: String!) {
  getActiveEvents(shopId: $shopId) {
    id name brief detail venue date price images capacity shopId createdAt
    options { id name values { id name extraAmount } }
  }
}
```

### Get Event
```graphql
query Event($eventId: String!) {
  getEvent(eventId: $eventId) {
    id name brief detail venue date price images capacity options { id name values { id name extraAmount } }
  }
}
```

### Create Event
```graphql
mutation CreateEvent(
  $shopId: String! $images: [String!]! $name: String! $brief: String! $detail: String!
  $venue: String $date: String! $price: String $options: [ProductOptionsInput!]! $capacity: Int
) {
  createEvent(shopId: $shopId, images: $images, name: $name, brief: $brief, detail: $detail, venue: $venue, date: $date, price: $price, options: $options, capacity: $capacity)
}
```
`price` is String (e.g. `"15.00"`). `date` is ISO 8601 datetime.

---

## Blog Layout (CMS)

Control how the `/blog` page looks. See [admin/shop-layout.md](../admin/shop-layout.md) ("Blog page") for the full JSON — keys are **snake_case** and the write replaces the whole blog layout.

Quick reference:
```graphql
mutation UpdateBlogLayout($shopId: String!, $blogLayout: JSON!) {
  updateShopBlogLayout(shopId: $shopId, blogLayout: $blogLayout) {
    id shopId
    blogLayout { template showFeatured postsPerPage showSidebar showCategories showSearch showAuthor showDate showExcerpt heroTitle heroSubtitle }
  }
}
```

Templates: `magazine_grid` / `card_grid` render a grid; `classic_list` / `minimal_list` / `custom` render a list. The storefront honours only `template`, `hero_title`, `hero_subtitle`, `show_date` and `show_excerpt`; the other flags are stored but unused.

---

> **Custom Pages**: For standalone pages (About, Terms, Contact, FAQ), see [admin/pages.md](../admin/pages.md) — those use `createPage`/`updatePage`/`deletePage` and are separate from blog posts and events.

## Agent Notes
- **Posts vs Events**: Posts are blog articles. Events have a date, venue, and optional ticket price/capacity.
- **Images**: Upload first via `POST ${RIVERBASE_API_URL%/graphql}/uploads/s3`, then pass the returned URLs.
- **Blog layout**: To customize the blog page appearance (grid vs list, hero title, date and excerpt), use `updateShopBlogLayout` documented in [admin/shop-layout.md](../admin/shop-layout.md).
