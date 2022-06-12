# Rails

**约定大于配置**

Pages & Assets

Dynamic pages: Routes / Restful / Inheritance

Assets: Asset Pipeline / Minify / Combine / Webpack / babel etc

## Dynamic Pages

### Routes

- Http GET/POST/PUT/DELETE

- Custom routes

- Easy to refactor

### Filter

- before_filter
- around_filter
- after_filter

### Cache

- Page cache
- Fragment cache

### Response

- HTML
- JSON
- XML
- etc.

## ActiveRecord

### Powerful ORM

```ruby
User.find(2)

User.find_by(email: 'code@111.com')

User.where(membership: 'gold').page(2).per_page(10).order("id desc").limit(10)

user.blogs

#etc
```

### Triggers

- before_save
- after_save
- after_destory
- after_create
- etc.

## Assets

- Javascript/CSS minify and combine
- Assets URL
- ETag
- SASS
- CoffeeScript

## Testing

Unit Test: rspect / minitest

Smoke Test / Continuous Integration: cucumber / selenium

## Script & Automation

- rails c
- rails g
- rails s
- etc.