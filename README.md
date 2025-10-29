# rails_gym

## 1.

- create a directory, initialize it on github. `rails_gym`
- The `rails new` command creates a directory, so I have to adapt it a little to populate an existing directory:
  - `rails new . --database=sqlite3`
  - `bin/rails db:create`
  - test the server by running `bin/rails s`

## 2. create db tables

  ```
  # Core tables
bin/rails g model Exercise name:string notes:text
bin/rails g model MuscleGroup name:string:uniq
bin/rails g model SessionType name:string:uniq

# Join tables (no standalone id needed)
bin/rails g model ExerciseMuscleGroup exercise:references muscle_group:references
bin/rails g model ExerciseSessionType exercise:references session_type:references

# Logged sets
bin/rails g model SetEntry exercise:references performed_on:date reps:integer weight:decimal{6,2} rpe:decimal{3,1} notes:text

bin/rails db:migrate

```

## 3. Routes

Open config/routes.rb and replace the contents with:

```
Rails.application.routes.draw do
  root "exercises#index"

  resources :muscle_groups
  resources :session_types

  resources :exercises do
    resources :set_entries, only: [:create, :destroy]
  end
end
```

## 4. Controllers & views (no new DB tables)

```
bin/rails g scaffold_controller MuscleGroup name:string
bin/rails g scaffold_controller SessionType name:string
bin/rails g scaffold_controller Exercise name:string notes:text
bin/rails g scaffold_controller SetEntry exercise:references performed_on:date reps:integer weight:decimal rpe:decimal notes:text
```

## 5. Exercise form: add the many-to-many checkboxes

Edit app/views/exercises/_form.html.erb so it permits tagging:

```
<%= form_with(model: exercise, local: true) do |f| %>
  <% if exercise.errors.any? %>
    <div id="error_explanation">
      <h2><%= pluralize(exercise.errors.count, "error") %> stopped this exercise from saving:</h2>
      <ul><% exercise.errors.full_messages.each { |m| %><li><%= m %></li><% } %></ul>
    </div>
  <% end %>

  <div>
    <%= f.label :name %>
    <%= f.text_field :name, autofocus: true %>
  </div>

  <div>
    <%= f.label :notes %>
    <%= f.text_area :notes %>
  </div>

  <hr>
  <div>
    <strong>Muscle groups</strong><br>
    <%= f.collection_check_boxes :muscle_group_ids, MuscleGroup.order(:name), :id, :name %>
  </div>

  <div style="margin-top: 8px;">
    <strong>Session types</strong><br>
    <%= f.collection_check_boxes :session_type_ids, SessionType.order(:name), :id, :name %>
  </div>

  <div style="margin-top: 12px;">
    <%= f.submit %>
  </div>
<% end %>
```

## 6 Permit the association IDs in the controller

Edit app/controllers/exercises_controller.rb → update exercise_params:

```
def exercise_params
  params.require(:exercise).permit(
    :name, :notes,
    muscle_group_ids: [],
    session_type_ids: []
  )
end
```

## 7 Exercise show page: quick set logging

Replace app/views/exercises/show.html.erb with:

```
<p id="notice"><%= notice %></p>

<h1><%= @exercise.name %></h1>
<p><%= simple_format(@exercise.notes) %></p>

<p>
  <strong>Muscles:</strong>
  <%= @exercise.muscle_groups.order(:name).pluck(:name).join(", ").presence || "-" %><br>
  <strong>Session types:</strong>
  <%= @exercise.session_types.order(:name).pluck(:name).join(", ").presence || "-" %>
</p>

<p>
  <%= link_to "Edit", edit_exercise_path(@exercise) %> |
  <%= link_to "Back", exercises_path %>
</p>

<hr>

<h2>Log a set</h2>
<%= form_with model: [@exercise, SetEntry.new], local: true do |f| %>
  <%= f.hidden_field :exercise_id, value: @exercise.id %>
  <%= f.date_field :performed_on, value: Date.today %>
  <%= f.number_field :reps, min: 1, placeholder: "Reps" %>
  <%= f.number_field :weight, step: "0.25", placeholder: "Weight (kg)" %>
  <%= f.number_field :rpe, step: "0.5", placeholder: "RPE (opt.)" %>
  <%= f.text_field   :notes, placeholder: "Notes (opt.)" %>
  <%= f.submit "Add set" %>
<% end %>

<h3>Recent sets</h3>
<ul>
  <% @exercise.set_entries.order(performed_on: :desc, created_at: :desc).limit(20).each do |s| %>
    <li>
      <%= s.performed_on %> — <%= s.reps %> reps @ <%= s.weight %> kg
      <% if s.rpe.present? %>(RPE <%= s.rpe %>)<% end %>
      <% if s.notes.present? %> — <%= s.notes %><% end %>
      — <%= link_to "delete",
            exercise_set_entry_path(@exercise, s),
            data: { turbo_method: :delete, confirm: "Delete this set?" } %>
    </li>
  <% end %>
</ul>
```

## 8 SetEntriesController: make create/destroy work

Edit app/controllers/set_entries_controller.rb:

```
class SetEntriesController < ApplicationController
  before_action :set_exercise

  def create
    @set_entry = @exercise.set_entries.new(set_entry_params)
    if @set_entry.save
      redirect_to @exercise, notice: "Set logged."
    else
      redirect_to @exercise, alert: @set_entry.errors.full_messages.to_sentence
    end
  end

  def destroy
    set_entry = @exercise.set_entries.find(params[:id])
    set_entry.destroy
    redirect_to @exercise, notice: "Set removed."
  end

  private

  def set_exercise
    @exercise = Exercise.find(params[:exercise_id])
  end

  def set_entry_params
    params.require(:set_entry).permit(:performed_on, :reps, :weight, :rpe, :notes)
  end
end
```

## 9 define relations in the models:

```
1) Set up the associations
app/models/exercise.rb
class Exercise < ApplicationRecord
  has_many :exercise_muscle_groups, dependent: :destroy
  has_many :muscle_groups, through: :exercise_muscle_groups

  has_many :exercise_session_types, dependent: :destroy
  has_many :session_types, through: :exercise_session_types

  has_many :set_entries, dependent: :destroy

  validates :name, presence: true
end
app/models/muscle_group.rb
class MuscleGroup < ApplicationRecord
  has_many :exercise_muscle_groups, dependent: :destroy
  has_many :exercises, through: :exercise_muscle_groups

  validates :name, presence: true, uniqueness: true
end
app/models/exercise_muscle_group.rb
class ExerciseMuscleGroup < ApplicationRecord
  belongs_to :exercise
  belongs_to :muscle_group
end
(Same pattern for session types:)
app/models/session_type.rb
class SessionType < ApplicationRecord
  has_many :exercise_session_types, dependent: :destroy
  has_many :exercises, through: :exercise_session_types

  validates :name, presence: true, uniqueness: true
end
app/models/exercise_session_type.rb
class ExerciseSessionType < ApplicationRecord
  belongs_to :exercise
  belongs_to :session_type
end
```

## 10 Permit the IDs in the controller

Because your form uses collection_check_boxes :muscle_group_ids, you must permit an array of IDs:
```
app/controllers/exercises_controller.rb
def exercise_params
  params.require(:exercise)
        .permit(:name, :notes, muscle_group_ids: [], session_type_ids: [])
end
```

## 11 Tailwind

1. 
```
bundle add tailwindcss-rails
bin/rails tailwindcss:install
rm app/assets/stylesheets/application.css 
```

2. add simple layout shell:

```
<!DOCTYPE html>
<html>
  <head>
    <title>Rails Gym</title>
    <meta name="viewport" content="width=device-width,initial-scale=1" />
    <%= csrf_meta_tags %>
    <%= csp_meta_tag %>
    <%= stylesheet_link_tag "application", "data-turbo-track": "reload" %>
    <%= javascript_importmap_tags %>
  </head>

  <body class="bg-gray-50 text-gray-900">
    <header class="bg-white border-b">
      <div class="max-w-4xl mx-auto px-4 py-4 flex items-center justify-between">
        <h1 class="text-xl font-semibold"><%= link_to "Rails Gym", root_path, class: "hover:underline" %></h1>
        <nav class="text-sm space-x-4">
          <%= link_to "Exercises", exercises_path, class: "hover:underline" %>
          <%= link_to "Muscles", muscle_groups_path, class: "hover:underline" %>
          <%= link_to "Session types", session_types_path, class: "hover:underline" %>
        </nav>
      </div>
    </header>

    <main class="max-w-4xl mx-auto px-4 py-6">
      <% if notice %>
        <div class="mb-4 rounded-lg bg-green-50 text-green-800 px-4 py-3 border border-green-200"><%= notice %></div>
      <% end %>
      <% if alert %>
        <div class="mb-4 rounded-lg bg-red-50 text-red-800 px-4 py-3 border border-red-200"><%= alert %></div>
      <% end %>

      <%= yield %>
    </main>
  </body>
</html>
```

3. Make the forms tidy

```
<%= form_with(model: exercise, class: "space-y-6") do |f| %>
  <% if exercise.errors.any? %>
    <div class="rounded-lg bg-red-50 border border-red-200 p-4">
      <h2 class="font-semibold text-red-800 mb-2">
        <%= pluralize(exercise.errors.count, "error") %> prevented saving:
      </h2>
      <ul class="list-disc pl-5 text-sm text-red-700">
        <% exercise.errors.full_messages.each do |m| %><li><%= m %></li><% end %>
      </ul>
    </div>
  <% end %>

  <div>
    <%= f.label :name, class: "block text-sm font-medium mb-1" %>
    <%= f.text_field :name, class: "w-full rounded-md border-gray-300 focus:border-indigo-500 focus:ring-indigo-500" %>
  </div>

  <div>
    <%= f.label :notes, class: "block text-sm font-medium mb-1" %>
    <%= f.text_area :notes, rows: 3, class: "w-full rounded-md border-gray-300 focus:border-indigo-500 focus:ring-indigo-500" %>
  </div>

  <div class="grid md:grid-cols-2 gap-6">
    <div>
      <div class="text-sm font-semibold mb-2">Muscle groups</div>
      <div class="grid grid-cols-2 gap-y-2">
        <%= f.collection_check_boxes :muscle_group_ids, MuscleGroup.order(:name), :id, :name do |cb| %>
          <label class="inline-flex items-center gap-2">
            <%= cb.check_box(class: "rounded border-gray-300 text-indigo-600 focus:ring-indigo-500") %>
            <span class="text-sm"><%= cb.text %></span>
          </label>
        <% end %>
      </div>
    </div>

    <div>
      <div class="text-sm font-semibold mb-2">Session types</div>
      <div class="grid grid-cols-2 gap-y-2">
        <%= f.collection_check_boxes :session_type_ids, SessionType.order(:name), :id, :name do |cb| %>
          <label class="inline-flex items-center gap-2">
            <%= cb.check_box(class: "rounded border-gray-300 text-indigo-600 focus:ring-indigo-500") %>
            <span class="text-sm"><%= cb.text %></span>
          </label>
        <% end %>
      </div>
    </div>
  </div>

  <div class="flex items-center gap-3">
    <%= f.submit class: "inline-flex items-center px-4 py-2 rounded-md bg-indigo-600 text-white hover:bg-indigo-700" %>
    <%= link_to "Cancel", exercises_path, class: "text-sm text-gray-600 hover:underline" %>
  </div>
<% end %>
```

4. Nicer show page actions:

```app/views/exercises/show.html.erb:
<div class="flex items-center justify-between mb-4">
  <h1 class="text-2xl font-semibold"><%= @exercise.name %></h1>
  <div class="flex gap-2">
    <%= link_to "Edit", edit_exercise_path(@exercise),
        class: "px-3 py-1.5 rounded-md border border-gray-300 hover:bg-gray-50" %>
    <%= link_to "Back", exercises_path,
        class: "px-3 py-1.5 rounded-md border border-gray-300 hover:bg-gray-50" %>
  </div>
</div>
```
