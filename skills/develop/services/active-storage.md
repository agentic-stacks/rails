# Active Storage

Active Storage handles file uploads, variant transformations, and cloud storage. Attachments are tracked in three database tables and files are stored in a configurable service backend.

## Install

```bash
bin/rails active_storage:install
bin/rails db:migrate
# Creates: active_storage_blobs, active_storage_attachments, active_storage_variant_records
```

## Attach Files to a Model

```ruby
# app/models/user.rb
class User < ApplicationRecord
  has_one_attached :avatar
end

# app/models/article.rb
class Article < ApplicationRecord
  has_one_attached :cover_image
  has_many_attached :documents
end
```

`has_one_attached` — a single file.
`has_many_attached` — a collection of files.

Neither adds a column to the model table; associations live in the Active Storage tables.

## Forms

```erb
<%# app/views/users/edit.html.erb %>
<%= form_with model: @user do |f| %>
  <%= f.label :avatar %>
  <%= f.file_field :avatar %>

  <%= f.submit "Save" %>
<% end %>
```

For `has_many_attached`, add `multiple: true`:

```erb
<%= f.file_field :documents, multiple: true %>
```

## Strong Parameters

```ruby
# app/controllers/users_controller.rb
def user_params
  params.require(:user).permit(:name, :email, :avatar)
end

# For has_many_attached:
def article_params
  params.require(:article).permit(:title, documents: [])
end
```

## Attaching in Code

```ruby
# Attach a file from disk
user.avatar.attach(
  io: File.open("/path/to/photo.jpg"),
  filename: "photo.jpg",
  content_type: "image/jpeg"
)

# Attach from a URL (Rails 7.1+)
user.avatar.attach(
  io: URI.open("https://example.com/photo.jpg"),
  filename: "photo.jpg"
)

# Attach from an uploaded params hash (controller)
user.avatar.attach(params[:user][:avatar])
```

## Checking and Purging

```ruby
user.avatar.attached?           # => true / false
user.avatar.purge               # delete synchronously
user.avatar.purge_later         # delete via Active Job (preferred)

article.documents.count         # => 3
article.documents.purge_later
```

## Displaying Attachments

```erb
<%# Show an image %>
<% if @user.avatar.attached? %>
  <%= image_tag @user.avatar %>
<% end %>

<%# Link to download a document %>
<%= link_to "Download report", rails_blob_path(@article.report, disposition: :attachment) %>

<%# Inline display %>
<%= link_to "View PDF", rails_blob_path(@article.report, disposition: :inline) %>
```

## Variants (Image Transformations)

Variants require either the `image_processing` gem (libvips or ImageMagick) or the built-in MiniMagick adapter.

**Gemfile:**

```ruby
gem "image_processing", "~> 1.2"
```

**Declare variants on the model:**

```ruby
# app/models/user.rb
class User < ApplicationRecord
  has_one_attached :avatar do |attachable|
    attachable.variant :thumb,  resize_to_limit: [100, 100]
    attachable.variant :medium, resize_to_limit: [300, 300]
  end
end
```

**Use in views:**

```erb
<%= image_tag @user.avatar.variant(:thumb) %>
<%= image_tag @user.avatar.variant(:medium) %>
```

**Inline variant** (without pre-declaration):

```erb
<%= image_tag @user.avatar.variant(resize_to_limit: [200, 200]) %>
```

Variants are processed on first request and cached as `active_storage_variant_records`.

## Analyzing Files

Active Storage enqueues `AnalyzeJob` automatically on attach. Metadata is stored on the blob:

```ruby
user.avatar.blob.metadata
# => { "width" => 1200, "height" => 800, "analyzed" => true }

user.avatar.blob.analyzed?         # => true
user.avatar.content_type           # => "image/jpeg"
user.avatar.byte_size              # => 204800
user.avatar.filename.to_s          # => "photo.jpg"
```

## Storage Services

Configure services in `config/storage.yml`:

```yaml
# config/storage.yml
local:
  service: Disk
  root: <%= Rails.root.join("storage") %>

amazon:
  service: S3
  access_key_id: <%= Rails.application.credentials.dig(:aws, :access_key_id) %>
  secret_access_key: <%= Rails.application.credentials.dig(:aws, :secret_access_key) %>
  region: us-east-1
  bucket: my-app-production

google:
  service: GCS
  project: my-gcp-project
  credentials: <%= Rails.root.join("config/gcs.json") %>
  bucket: my-app-production

azure:
  service: AzureStorage
  storage_account_name: myaccount
  storage_access_key: <%= Rails.application.credentials.dig(:azure, :storage_access_key) %>
  container: my-app-production
```

Point each environment at a service:

```ruby
# config/environments/development.rb
config.active_storage.service = :local

# config/environments/production.rb
config.active_storage.service = :amazon
```

Install the S3 gem when using Amazon:

```ruby
# Gemfile
gem "aws-sdk-s3", require: false
gem "google-cloud-storage", "~> 1.11", require: false  # GCS
gem "azure-storage-blob", "~> 2.0", require: false     # Azure
```

## Direct Upload

Direct upload lets the browser send the file straight to the storage service, bypassing your Rails server. Requires the `@rails/activestorage` JavaScript package.

**Enable in the form:**

```erb
<%= form_with model: @article do |f| %>
  <%= f.file_field :cover_image, direct_upload: true %>
<% end %>
```

**Import and start the library** (if using importmap):

```javascript
// app/javascript/application.js
import * as ActiveStorage from "@rails/activestorage"
ActiveStorage.start()
```

The browser:
1. Requests a presigned URL from your Rails app (`/rails/active_storage/direct_uploads`)
2. Uploads directly to S3 / GCS / Azure using that URL
3. Submits the signed blob ID in the form

## Proxying vs Redirecting

By default, Rails redirects `rails_blob_path` to a short-lived signed URL on the storage service.

To proxy files through your Rails server (hides cloud URLs, allows CDN caching):

```ruby
# config/environments/production.rb
config.active_storage.resolve_model_to_route = :rails_storage_proxy
```

Or use the proxy URL helper directly:

```erb
<%= image_tag rails_storage_proxy_path(@user.avatar) %>
```

## Seeding with Attachments in Tests and Seeds

```ruby
# db/seeds.rb
user = User.create!(name: "Alice", email: "alice@example.com")
user.avatar.attach(
  io: File.open(Rails.root.join("test/fixtures/files/avatar.jpg")),
  filename: "avatar.jpg",
  content_type: "image/jpeg"
)
```

Use `fixture_file_upload` in tests:

```ruby
# test/models/user_test.rb
test "attaches avatar" do
  user = users(:alice)
  user.avatar.attach(fixture_file_upload("avatar.jpg", "image/jpeg"))
  assert user.avatar.attached?
end
```
