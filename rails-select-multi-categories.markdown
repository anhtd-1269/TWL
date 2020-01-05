---
layout: slide
title:  Select muli categories with rails + jquery
description: Select muli categories with rails + jquery
theme: league
author: Hungnv950
transition: fade
date: 2019-12-14
---

<section data-markdown style="text-align: left;">

## Select muli categories

## sử dụng rails + jquery

- Author: [hungnv950](https://github.com/hungnv950)

- Repo: [https://github.com/Hungnv950/rails-select-multi-categories](https://github.com/Hungnv950/rails-select-multi-categories)

- Representor: [https://hungnv950.github.io/2019/12/14/slect-multy-categories/#/](https://hungnv950.github.io/2019/12/14/slect-multy-categories/#/)
</section>


<section data-markdown style="text-align: left;">

  ### Vấn đề

  - Hầu hết các dự án out source và dự án rails cho thị trường Nhật nói chung việc làm việc với lựa chọn dữ liệu dùng checkbox là rất phổ biến.

  - ![problem](https://fluidsurveys.com/wp-content/uploads/2009/11/56.png)
  - Lựa chọn categories cho người dùng
</section>

<section data-markdown style="text-align: left;">

  ### Cách giải quyết

  ![Vào code dự án cũ để xem lại.](https://as2.ftcdn.net/jpg/01/70/81/05/500_F_170810520_qTnUe8qFYZo6Rc9rcBpXuUvYDl4v1pDi.jpg "Codding")
</section>

<section data-markdown style="text-align: left;">
  <p> Vào code dự án cũ để xem lại. </p>
  <br/>

  ![Vào code dự án cũ để xem lại.](https://letweb.net/wp-content/uploads/2018/07/Source-Code-M%C3%A3-ngu%E1%BB%93n-l%C3%A0-g%C3%AC.png)
</section>


<section data-markdown style="text-align: left;">

  ![alt text](https://res.cloudinary.com/drdoqfhly/image/upload/v1530887094/gg-1_synrgy.jpg)
</section>

<section data-markdown style="text-align: left;">

  Tạo 1 base code:
   - Đơn giản
   - Dễ hiểu
   - Đúng vấn đề
   - Có thể tái sử dụng
</section>

<section style="text-align: left;">
  <h3> Các bước cần làm </h3>
  <ol>
    <li>Tìm hiểu bài toán</li>
    <li>Cấu trúc database</li>
    <li>Khởi tạo model</li>
    <li>Xây dựng form checkbox</li>
    <li>Lưu trữ dữ liệu</li>
    <li>Lấy giá trị</li>
  </ol>
</section>

<section data-markdown style="text-align: left;">

  ### Tìm hiểu bài toán

  ![problem](https://fluidsurveys.com/wp-content/uploads/2009/11/56.png)
</section>

<section data-markdown style="text-align: left;">

  ### Cấu trúc database

  ![Select multi categories db](https://hungnv950.github.io/assets/images/multi-categories-db.png)
</section>

<section style="text-align: left;">
  <h3> Khởi tạo model </h3>

  <pre style="width: 100%;">
  <code data-trim data-noescape>
    # frozen_string_literal: true
    class User < ApplicationRecord
      has_many :users_categories, foreign_key: :user_id, dependent: :destroy
    end
	</code></pre>

  <pre style="width: 100%;"><code data-trim data-noescape>
    # frozen_string_literal: true
    class Category < ApplicationRecord
      has_many :users_categories, foreign_key: :category_id,
        dependent: :destroy
    end
	</code></pre>

  <pre style="width: 100%;"><code data-trim data-noescape>
    # frozen_string_literal: true
    class UsersCategory < ApplicationRecord
      belongs_to :user, foreign_key: :user_id
      belongs_to :category, class_name: Category.name
    end
	</code></pre>
</section>

<section data-markdown style="text-align: left;">

  ### Xây dựng form checkbox

  Yêu cầu:
  1. Generate form checkbox với dữ liệu category
  2. Khi lựa chọn vào "other content" sẽ xuất hiện input để nhập dữ liệu
  3. Trả lại đúng data khi validate sai
</section>

<section data-markdown style="text-align: left;">

  #### 1. Generate form checkbox với dữ liệu category

  Sử dụng nested attributes:
    - accepts_nested_attributes_for
    - fields_for
</section>

<section data-markdown style="text-align: left;">

  - Thêm đoạn dưới vào model user:

    ```
    accepts_nested_attributes_for :users_categories, allow_destroy: true
    ```
</section>

<section style="text-align: left;">
  - Trong users_controller/new tiến hành build giá trị cho users_categories:
    <pre style="width: 100%;"><code data-trim data-noescape>
    def new
      @user = User.new
      @category_options =
        Category.all.map do |category|
          UsersCategory.new category: category, user: @user
        end
    end
    </code></pre>
</section>

<section style="text-align: left;">
  Hiển thị dữ liệu trên view với fields_for:
  <br/>
  fields_for(record_name, record_object = nil, options = {}, &block)
  <br/>

  check_box(object_name, method, options = {}, checked_value = "1", unchecked_value = "0")

  <pre style="width: 100%;"><code data-trim data-noescape>
<%= form_for @user do |f| %>
  <%= f.object.errors.full_messages %>
  username: <%= f.text_field :name %>
  categories:
    <%= f.fields_for :users_categories, @category_options do |ff| %>
      <% category_option = ff.object %>
      <%= ff.check_box :_destroy, {}, checked_value: "0", unchecked_value: "1" %>
      <%= ff.hidden_field :category_id %>
      <%= category_option.content %>
    <% end %>
  <%= f.submit %>
<% end %>
    </code></pre>
</section>

<section data-markdown style="text-align: left;">

  ```
  <%= ff.check_box :_destroy, {}, checked_value: "0", unchecked_value: "1" %>
  ```
  - Mặc định `check_box` sẽ để `checked_value` = 1 và `unchecked_value` = 0. Nhưng trong trường hợp này những giá trị được checked sẽ đại diện cho `_destroy` có nghĩa là những checkbox nào được tích sẽ bị xóa.
  - Vì thế nên chúng ta sẽ đổi lại giá trị mặc định `checked_value: "0"` và `unchecked_value: "1"` để giữ lại những `user_categories` được tick.
</section>

<section data-markdown style="text-align: left;">

  - Kết quả:
    ![Select multi categories db](https://hungnv950.github.io/assets/images/select-multi-categories-form-checkbox.png)
</section>

<section style="text-align: left;">
   <h4> 2. Khi lựa chọn vào "other content" sẽ xuất hiện input để nhập dữ liệu </h4>
   <p> - Ý tưởng: Đối với category có key_name = "other_content" sẽ generate ra input và ẩn đi, khi người dùng click vào `other_content` sẽ show và hide </p>
      <pre style="width: 100%;"><code data-trim data-noescape>
<%= form_for @user do |f| %>
  <%= f.object.errors.full_messages %>
  username: <%= f.text_field :name %>
  categories:
    <%= f.fields_for :users_categories, @category_options do |ff| %>
      <% users_categories = f.object.users_categories %>
      <% category_option = ff.object %>
        <%= ff.check_box :_destroy, {}, 0, 1 %>
        <%= ff.hidden_field :category_id %>
        <%= category_option.content %>

        # Thêm đoạn
        <% if category_option.key_name == "other_content" %>
          <%= ff.text_field :other_content, value: other_content,class: "js-other_content-field hidden" %>
        <% end %>

    <% end %>
  <%= f.submit %>
<% end %>
    </code></pre>
</section>

<section style="text-align: left;">
   <p> - Xử lý một chút JS đơn giản: </p>
      <pre style="width: 100%;"><code data-trim data-noescape>
  $(function() {
    # Xử lý với trường hợp người dùng click vào `other_category`
    $('.js-select-other_content').change(function () {
      var self = $(this),
          otherContent = $('.js-other_content-field');
      if (self.is(':checked')) {
        otherContent.removeClass('hidden');
      } else {
        otherContent.addClass('hidden');
      }
    });
  });
    </code></pre>
</section>

<section data-markdown style="text-align: left;">

  #### 2. Khi lựa chọn vào "other content" sẽ xuất hiện input để nhập dữ liệu

  - Kết quả:
    ![Select multi categories db](https://hungnv950.github.io/assets/images/select-multi-categories-click-other.gif)
</section>

<section data-markdown style="text-align: left;">

  ### Lưu trữ dữ liệu
</section>

<section data-markdown style="text-align: left;">

  - Permit data
  - Validate
  - Điền data cũ vào form nếu validate sai
</section>

<section style="text-align: left;">
  <p> 1.Permit data </p>
  <pre style="width: 100%;"><code data-trim data-noescape>
  def user_params
    params.require(:user).permit(
      :name,
      users_categories_attributes: [:id, :category_id, :other_content, :_destroy]
    )
  end
    </code></pre>
</section>

<section style="text-align: left;">
  <p> 2. Validate </p>
  - số lượng category tối thiểu<br>
  - validate nội dung của "other content" khi chọn checkbox này
</section>

<section style="text-align: left;">
  <pre style="width: 100%;"><code data-trim data-noescape>
    # user.rb
    MIN_SIZE = 1
    # Validate số lượng tối thiểu categories
    validates :users_categories, length: {minimum: MIN_SIZE}
  </code></pre>

  Hoặc sử dụng custom validate:
    <pre style="width: 100%;"><code data-trim data-noescape>
    # user.rb
    ### Validate số lượng tối thiểu categories
    MIN_SIZE = 1
    validate :validate_users_categories

    private
    def validate_users_categories
      errors.add(:users_categories, :minsize) if(users_categories.size < MIN_SIZE)
    end
  </code></pre>
</section>


<section style="text-align: left;">
  <pre style="width: 100%;"><code data-trim data-noescape>
# users_category.rb
# Validate nội dung khi chọn "other_content"
validates :other_content, presence: true, if: lambda { key_name == "other_content"}
  </code></pre>
  Hoặc sử dụng custom validate:
    <pre style="width: 100%;"><code data-trim data-noescape>
# users_category.rb
# Validate nội dung khi chọn "other_content"
validate :validate_other_content
private
def validate_other_content
  errors.add(:other_content, :blank) if(key_name == OTHER_CONTENT && other_content.blank?)
end
  </code></pre>
</section>

<section data-markdown style="text-align: left;">

  #### 3. Điền giá trị cũ vào form nếu validate sai

  Có 2 trường hợp cần xử lý:
    - Checkbox
    - Nội dung của "other content" nếu người dùng lựa chọn ô này.
</section>

<section data-markdown style="text-align: left;">

  - Checkbox
    Ý tưởng: Khi submit dữ liệu lên controller, @user đã được gán giá trị khi "@user = User.new user_params". Khi này, object @user đã được khởi tạo và có association.
    ![Select multi categories db](https://hungnv950.github.io/assets/images/select-ml-ct-object-user.png)
  - Đây chính là những checkbox đã được checked
</section>

<section style="text-align: left;">

  <pre style="width: 100%;"><code data-trim data-noescape>
# new.html.rb
<%= form_for @user do |f| %>
<%= f.object.errors.full_messages %>
username: <%= f.text_field :name %>
categories:
  <%= f.fields_for :users_categories, @category_options do |ff| %>
    <% users_categories = f.object.users_categories %>
    <% category_option = ff.object %>
    # Tìm kiếm users_categories có chứa category_option hiện tại hay không
    <% is_selected = users_categories.map{|c| c.key_name}.include?(category_option.key_name) %>
    # Lấy giá trị của "Other content"
    <% other_content = users_categories.select{|c| c.key_name == "other_content"}.first&.other_content %>
      <%= ff.check_box :_destroy, {checked: is_selected, class: "#{'js-select-other_content' if category_option.key_name == 'other_content'}"}, 0, 1 %>
      <%= ff.hidden_field :category_id %>
      <%= category_option.content %>
      <% if category_option.key_name == "other_content" %>
        <%= ff.text_field :other_content, value: other_content,class: "js-other_content-field hidden" %>
      <% end %>
  <% end %>
<%= f.submit %>
<% end %>
  </code></pre>
</section>

<section data-markdown style="text-align: left;">

  ### Lấy giá trị
  Có 2 trường hợp:
    - Nếu user_category có key_name không phải "other content" thì sẽ lấy giá trị "content" từ bảng "category"
    - Nếu users_category có key_name là "other_content" sẽ tiến hành lấy "other_content" từ trong bảng users_category
</section>

<section style="text-align: left;">
  <pre style="width: 100%;"><code data-trim data-noescape>
# users_category.rb
def category_name
  key_name == OTHER_CONTENT ? other_content : content
end
  </code></pre>

  <pre style="width: 100%;"><code data-trim data-noescape>
# user.rb
def show
  @user = User.find params[:id]
  @categories = @user.users_categories.map{|c| c.category_name}.join(" ,")
end
  </code></pre>
</section>


<section data-markdown style="text-align: left;">

  ### Trường hợp edit

  - Đối với trường hợp edit, khi chúng ta chạy đoạn:

  ```
is_selected = users_categories.map{|c| c.category_key_name}.include?(category_option.category_key_name)
  ```

  thì sẽ phải thực hiện truy vấn thông qua database để lấy lại association. Khi đó khi chúng ta cập nhật thay đổi giá trị cho ô select, ô đó sẽ luôn là giá trị cũ
</section>

<section data-markdown style="text-align: left;">

  ![Select multi categories db](https://hungnv950.github.io/assets/images/1.png)

</section>

<section data-markdown style="text-align: left;">

  ![Select multi categories db](https://hungnv950.github.io/assets/images/2.png)

</section>

<section data-markdown style="text-align: left;">

  ![Select multi categories db](https://hungnv950.github.io/assets/images/3.png)

</section>

<section data-markdown style="text-align: left;">

  ### Edit | Ý tưởng

  - Khi submit params lên chúng ta sẽ lọc được những params có category đã được chọn
  - Lọc và trả lại những id đó

  ```
  @selected_ids =
    users_categories_attributes.values.map {|c| c[:category_id] if c[:_destroy] == "0"}.compact
  ```

  - Kết quả: Mảng các categories đã selected. Ví dụ: ["1", "2"]

</section>

<section data-markdown style="text-align: left;">

  - Edit một chút trong view:

  ```
    selected_ids = @selected_ids ||
    users_categories.map{|c| c.category_id.to_s}
  ```

  ```
  is_selected = selected_ids.include?(category_option.category_id.to_s)
  ```

  - Kết quả: Mảng các categories đã selected. Ví dụ: ["1", "2"]
</section>



<section data-markdown style="text-align: left;">

  ### Lưu ý

  - Khi cập nhật dữ liệu cần phải permit id của nested params để tránh tình trạng tạo thêm bản ghi mới giống bản ghi cũ.

  - Trong file edit.html.slim thêm hidden_field cho id của users_category.

  ```
  <%= ff.hidden_field :id, value: users_categories.select{|c| c.key_name == category.key_name}.first&.id %>
  ```

  - Về bản chất neseted_attributes đã hỗ trợ tự điền id vào form nested nhưng trong trường hợp này giá trị của nested form được build ra không phải là giá trị trong quan hệ mà là giá trị của object `@category_options` chúng ta truyền vào nên cần phải lọc ra như trên.
  - Vị trí của categories: Thông thường vị trí của trường `other_content` sẽ là ở cuối danh sách. Nhưng không hẳn master data nào cũng chuẩn hoặc category được sinh tự động. Có một mẹo là chúng ta sẽ luôn để `other_content` ở đầu tiên của mảng (bản ghi đầu tiên) và dùng hàm `rotate(1)` để dịch bản ghi đó xuống cuối mảng

  ```
  a = [ "a", "b", "c", "d" ]
  a.rotate         #=> ["b", "c", "d", "a"]
  ```

  - Tìm hiểu kĩ design để khi import master
</section>

<section data-markdown style="text-align: left;">

  ### End.

  #### Cảm ơn mọi người đã lắng nghe!

</section>
