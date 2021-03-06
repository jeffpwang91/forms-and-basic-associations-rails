# Forms And Basic Associations Rails

## Objectives

1. Populate select options based on association options.
2. Assign a FK based on an input box value directly through mass assignment. (post[category_id])
3. Define a belongs_to association writer.
4. Build a form field that will delegate to a belongs_to association writer. (post#category_name=) through controller mass assignment.
5. Define a has_many association writer.
6. Build a form field that will delegate to a has_many association writer. (owner#pet_names=) through controller mass assignment.

## The problem

Let's say we have a simple blogging system. Our models are Post and Category. A Post belongs_to a Category.

```
# app/models/post.rb
class Post < ActiveRecord::Base
  belongs_to :category
end

# app/models/category.rb
class Category < ActiveRecord::Base
  has_many :posts
end
```

When a user creates a post, how will they specify what category it belongs to?

## Using the category ID

As a first pass, we might build a form like this:

```
<%= form_for @post do |f| %>
  <label>Category: <input type="text" name="post[category_id]"></label>
  <textarea name="post[content]"></textarea>
<% end %>
```

This will work if we wire up our `PostsController` with the right parameters:

```
class PostsController < ApplicationController
  def create
    Post.create(post_params)
  end

  private

  def post_params
    params.require(:post).permit(:category_id, :content)
  end
end
```

But as a user experience, this is miserable. I have to know the id of the category I
want to use. As a user, it is very unlikely that I know this or want to.

We could rewrite our controller to accept a `category_name` instead of an id:

```
class PostsController < ApplicationController
  def create
    category = Category.find_or_create_by(name: params[:category_name])
    Post.create({content: params[:content], category: category})
  end
end
```

But we'll have to do this anywhere we want to set the category for a Post. When we're
setting a Post's categories, the one thing we know we have is a Post object. What if we could
move this logic to the model?

Specifically, what if we gave the Post model a `category_name` attribute?

## You can define your own attributes on models

Since our ActiveRecord models are still just Ruby classes, we can define our own set
methods:

```
# app/models/post.rb
class Post < ActiveRecord::Base
   def category_name=(name)
     self.category = Category.find_or_create_by(name: name)
   end
end
```

Now we can set `category_name` on a post. We can do it when creating a post too, so our
controller becomes quite simple again:

```
class PostsController < ApplicationController
  def create
    Post.create(post_params)
  end

  private

  def post_params
    params.require(:post).permit(:category_name, :content)
  end
end
```

Notice the difference—we're now accepting a category name, rather than a category id. Even though you don't have an ActiveRecord field for `category_name`, because there is a key in the `post_params` hash for `category_name` it still calls the `category_name=` method. Oh hey! we created our own `category_name=` method! So convenient.

We can change the view as well now:

```
<%= form_for @post do |f| %>
  <label>Category: <input type="text" name="post[category_name]"></label>
  <textarea name="post[content]"></textarea>
<% end %>
```

Now the user can enter a category by name, a much friendlier experience.

## Selecting from existing categories

If we want to let the user pick from existing categories, we can use a the [collection_select]
helper to render a `<select>` tag:

```
<%= form_for @post do |f| %>
  <%= f.collection_select :category, Category.all, :id, :name %>
  <textarea name="post[content]"></textarea>
<% end %>
```

This will create a drop down selection input where the user can pick a category.

However, we've lost the ability for users to create their own categories.

That might be what you want. For example, the content management system for a magazine
would probably want to enforce that the category of an article is one of the sections
actually printed in the magazine.

In this case, I think we want to give users the flexibility to either create a new category,
or pick an existing one. What we want is autocompletion, which we can get with a [datalist]:

```
<%= form_for @post do |f| %>
  <%= f.text_field :category, list: "categories_autocomplete" %>
  <datalist id="categories_autocomplete">
    <%= Category.all.each do |category| %>
      <option value="<%= category.name %>">
    <% end %>
  </datalist>
  <textarea name="post[content]"></textarea>
<% end %>
```

Data lists are a new select type in the HTML5 spec that allows for easy autocomplete. Check out [this codepen](http://codepen.io/matt-west/pen/jKnzG) for how they work!

## Updating multiple rows

Let's think about the reverse association. Categories have many posts. 

```ruby
# app/models/category.rb
class Category < ActiveRecord::Base
  has_many :posts
end
```

Given a category, how do we let a user specify many different posts to categorize? We can't do it with just one `<select>` because we can have many posts in that category.

### Using array parameters

Rails uses a [naming convention] to let you submit an array of values to a controller.

If you put this in a view,

```
<%= form_for @category do |f| %>
  <input name="post_ids[]">
  <input name="post_ids[]">
  <input name="post_ids[]">
<% end %>
```

When the form is submitted, your controller will have access to a `post_ids` param, which
will be an array of strings.

We can write a setter method for this, just like we did for `category_name`:

```
# app/models/category.rb
class Category < ActiveRecord::Base
   def post_ids=(ids)
     ids.each do |id|
       post = Post.find(id)
       posts << post
     end
   end
end
```

Now we can use the same wiring in the controller to set `post_ids` from `params`:

```
# app/controllers/categories_controller.rb
class CategoriesController < ApplicationController
  def create
    Category.create(category_params)
  end

  private

  def category_params
    params.require(:category).permit(:name, :post_ids)
  end
end
```

[collection_select]: http://apidock.com/rails/ActionView/Helpers/FormOptionsHelper/collection_select
[naming convention]: http://guides.rubyonrails.org/v3.2.13/form_helpers.html#understanding-parameter-naming-conventions
[datalist]: https://developer.mozilla.org/en-US/docs/Web/HTML/Element/datalist

<p data-visibility='hidden'>View <a href='https://learn.co/lessons/forms-and-basic-associations-rails'>Forms and Basic Association </a> on Learn.co and start learning to code for free.</p>
