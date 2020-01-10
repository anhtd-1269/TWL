# Chủ đề: Tạo base import masterdata
## Mục lục
### 1. [Vấn đề thường gặp](#vấn-đề-thường-gặp)
### 2. [Tạo base import](#tao-base-import)
### 3. [Case đặc biệt & mẹo cần chú ý](#case-đặc-biệt-&-mẹo-cần-chú-ý)
### 4. [Test hiệu năng các phương pháp Import](#test-hiệu-năng-các-phương-pháp-import)
# Vấn đề thường gặp
## 1. Performance luôn là vấn đề lớn cần giải quyết trong các bài toán import số lượng lớn

Vấn đề này đã được đề cập nhiều, khi import 1 file, nếu chỉ là một master data, có tầm chục record, thì không đáng kể, ta có thể sử dụng cách đơn giản nhất là gọi hàm save truyền thống , tuy nhiên, mới dữ liệu vài chục nghìn, trăm nghìn hay vài triệu thì đó là câu chuyện khác.

Bạn có thể chạy lệnh import trong background job, để không ảnh hưởng đến tốc độ trải nghiệm trang web, nhưng nó vẫn sẽ ảnh hưởng đến tài nguyên hệ thống, khi mà cần quá nhiều thời gian và dữ liệu để thực hiện import.
Vì vậy, có một phương pháp import data với dữ liệu lớn sau:

Sử dụng gem activerecord-import Gem này hỗ trợ khả năng import thông qua việc vài method import cho **model**

Giả sử ta có model với:
``` Ruby
columns = [:comment, :start]
values = []
TIMES = 5

TIMES.times do
    values.push ["comment", 1]
end

Post.import columns, values
```
Gem hỗ trợ validation thông qua 2 option true, false

``` Ruby
Post.import columns, values, validate: false
```
sẽ giúp tăng hiệu năng import vì bỏ qua validation dữ liệu
## 2. Import dữ liệu có xử lý validation
Với các phương pháp import csv đều có 1 kết luận, nếu muốn tăng tốc độ import, thì phải bỏ qua validation. Vậy, chả lẽ ta phải import cả data sai vào db, diều đó là không ổn , sẽ mất tính toàn vẹn dữ liệu, và về sau sẽ rất mất công cập nhật là dữ liệu cho đúng

Vì vậy, trước khi import, ta tạo 1 service kiểm tra validation cho các row, bằng cách đơn giản là gọi hàm valid? cho từng object. Có 1 thủ thuật tôi thấy rất hay cho trường hợp kiểm tra này, đó là

``` Ruby
# thay vì viết
post1 = Post.new(comment: "comment1", star: 1)
post1.valid?
post2 = Post.new(comment: "comment2", star: 1)
post2.valid?

# thì viết
post = Post.new
post.assign_attributes comment: "comment1", star: 1
post.valid?

post.assign_attributes comment: "comment2", star: 1
post.valid?
```

lợi ích của cách này là giảm bớt số lượng lớn object phải khởi tạo, thay vì như cách cũ là từng comment thì tạo từng đấy object, thì ta tạo 1 object duy nhất và assign_attributes object theo từng row, như vậy sẽ tăng hiệu năng rất nhiều vì đã giảm bớt tài nguyên cần lưu trữ object.

## 3. Đặt vấn đề
  Với mong muốn xây dựng một base import master data để các dự án khác nhau có thể sử dụng lại, phù hợp với nhiều loại master data khác nhau, dễ mở rộng.
  => Với kinh nghiệm đúc rút qua các dự án đã làm, sau đây mình xin chia sẻ một base import master data với hy vọng có thể củng cố lại kiến thức cũng như phần nào giúp đỡ được mọi người khi gặp vấn đề tương tự.

# Tao base Import
## 1. Model
* Tao module M quan ly cac model import
``` Ruby
app/models/m.rb

module M
  def self.table_name_prefix
    "m_"
  end
end
```
* Tao model
``` Ruby
app/models/m/category.rb

class M::Category < ApplicationRecord
end
```

## 2. Service
* Import CSV service
``` Ruby
app/services/import_csv_service.rb

class ImportCsvService
  attr_reader :table_name, :csv_path, :validate

  def initialize table_name, csv_path, validate = true
    @table_name = table_name
    @csv_path = csv_path
    @validate = validate
  end

  def perform
    ActiveRecord::Base.transaction do
      model.delete_all
      # Xóa hết dữ liệu cũ trước khi import
      model.import records, batch_size: 10_000, validate: validate
      # batch_size là số lượng bản ghi lớn trong một lần import
      # validate là option true hay false cho việc xử lý validation khi import
    end
  rescue StandardError => e
    Rails.logger.error e.message
  end

  private
  def model
    @model ||= parse_table_name
  end

  def records
    @records ||= CSV.foreach(csv_path, headers: true).map do |row|
      model.new row.to_hash
    end
  end

  def parse_table_name
    table_name.gsub(/\Am_/, "m::_").classify.constantize
  end
end
```
* Import master data service
``` Ruby
app/services/import_master_data_service.rb

class ImportMasterDataService
  MASTER_TABLES = %w[m_categories m_xyz].freeze
  # Mỗi lần mở rộng thêm một loại csv mới bạn chỉ cần thêm nó ở đây, ví dụ m_zipcode

  def perform
    MASTER_TABLES.each do |table_name|
      csv_path = Rails.root.join "db", "seeds", "#{table_name}.csv"
      ImportCsvService.new(table_name, csv_path, false).perform
    end
  end
end
```
## 3. Viết Unit test cho import
Hiệu năng không chỉ là vấn đề cần giải quyết với import master data mà chúng ta cũng sẽ gặp trong quá trình viết unit test cho nó. Sẽ mất rất nhiều thời gian khi chạy test cho những file CSV master data có số lượng bản ghi lớn, lúc này chúng ta có thể làm như sau:
* Import master data service
``` Ruby
app/services/import_master_data_service.rb

class ImportMasterDataService
  MASTER_TABLES = %w[m_categories].freeze
  SLIM_TABLES_FOR_TEST = %w[m_du_lieu_lon].freeze
  # Tách riêng những CSV có số bản ghi lớn có thể ảnh hưởng đén hiệu năng test

  def perform
    MASTER_TABLES.each do |table_name|
      file_path = csv_path table_name
      ImportCsvService.new(table_name, file_path, false).perform
    end
  end

  private
  def csv_path table_name
    if import_slim_table? table_name
      # Nếu service đang chạy trong môi trường test sẽ import với file csv với số bản ghi phù hợp hơn để giảm thời gian chạy test
      Rails.root.join "spec", "fixtures", "csv", "#{table_name}_test.csv"
    else
      Rails.root.join "db", "seeds", "#{table_name}.csv"
    end
  end

  def import_slim_table? table_name
    # Kiểm tra service đang sử dụng trong môi trường nào
    Rails.env.test? && SLIM_TABLES_FOR_TEST.include?(table_name)
  end
end

```
* import_csv_service_spec.rb
``` Ruby
require "rails_helper"

RSpec.describe ImportCsvService, type: :service do
  describe "#perform" do
    let(:table_name) { "m_prefectures" }
    let(:csv_path) { Rails.root.join "db", "seeds", "#{table_name}.csv" }
    let(:service) { ImportCsvService.new table_name, csv_path, false }

    before { M::Prefecture.delete_all }

    it "should import data" do
      expect { service.perform }.to change(M::Prefecture, :count).by 47
    end
  end
end

```
* import master_data_service_spec.rb
``` Ruby
require "rails_helper"

RSpec.describe ImportMasterDataService, type: :service do
  describe "#perform" do
    before :all do
      DatabaseCleaner.clean_with :deletion
      ImportMasterDataService.new.perform
    end

    ImportMasterDataService::MASTER_TABLES.each do |table_name|
      it "should import data to #{table_name}" do
        model = table_name.gsub(/\Am_/, "m::_").classify.constantize
        expect(model.count).to be > 0
      end
    end
  end
end

```
## Case đặc biệt & mẹo cần chú ý
### 1. Tại sao cần thêm cột index cho file CSV
Để phòng trường hợp import dữ liệu mới sẽ mất tính toàn vẹn của dữ liệu cũ, chúng ta nên bổ sung một cột index, giá trị cột index này bằng với giá trị cột ID trong db, như vậy mỗi lần thêm giá trị mới thì ID của các giá trị trước đó sẽ không thay đổi tránh việc hỏng, mất dữ liệu cũ.
### 2. Trường hợp dữ liệu CSV có trường other
Ví dụ cụ thể cho trường hợp này là CSV category, spec dự án thường sẽ có phần chọn categoty other (khác) khi người dùng không nằm trong các danh mục mà hệ thống cung cấp, thông thường mục other này nằm cuối cùng trong danh sách chọn.

Trong file CSV khách hàng cung cấp thông thường trường Other sẽ nằm ở cuối cùng chúng ta cần đưa trường này lên đầu file CSV, điều này giúp chúng ta dễ dàng quản lý khi khách hàng bổ sung dữ liệu mới vào file csv.
Chi tiết thêm các bạn có thể tham khảo qua bài viết [Select muli categories](https://github.com/trungtq-0433/TWL/blob/master/rails-select-multi-categories.markdown)
## Test hiệu năng các phương pháp Import

```ruby
require "ar-extensions"

CONN = ActiveRecord::Base.connection
TIMES = 10000

def do_inserts
  TIMES.times {|i| Post.create comment: "comment #{i}", start: i}
end

def raw_sql
  TIMES.times {CONN.execute "INSERT INTO `posts` (`comment`, `star`, `updated_at`) VALUES('comment', 1, '2009-01-23 20:21:13')"}
end

def mass_insert
  inserts = []
  TIMES.times do
    inserts.push "('comment', 1, '2009-01-23 20:21:13')"
  end
  sql = "INSERT INTO posts (`comment`, `star`, `updated_at`) VALUES #{inserts.join(", ")}"
  CONN.execute sql
end

def activerecord_extensions_mass_insert validate = true
  columns = [:comment, :star]
  values = []
  TIMES.times do |i|
    values.push ["comment #{i}", i]
  end

  Post.import columns, values, {validate: validate}


puts "Testing various insert methods for #{TIMES} inserts\n"
puts "ActiveRecord without transaction:"
puts base = Benchmark.measure {do_inserts}

puts "ActiveRecord with transaction:"
puts bench = Benchmark.measure {ActiveRecord::Base.transaction{ do_inserts }}
puts sprintf("%2.2fx faster than base", base.real / bench.real)

puts "Raw SQL without transaction:"
puts bench = Benchmark.measure {raw_sql}
puts sprintf("%2.2fx faster than base", base.real / bench.real)

puts "Raw SQL with transaction:"
puts bench = Benchmark.measure {ActiveRecord::Base.transaction {raw_sql }}
puts sprintf("%2.2fx faster than base", base.real / bench.real)

puts "Single mass insert:"
puts bench = Benchmark.measure {mass_insert}
puts sprintf("%2.2fx faster than base", base.real / bench.real)

puts "ActiveRecord::Extensions mass insert:"
puts bench = Benchmark.measure {activerecord_extensions_mass_insert}
puts sprintf("%2.2fx faster than base", base.real / bench.real)

puts "ActiveRecord::Extensions mass insert without validations:"
puts bench = Benchmark.measure {activerecord_extensions_mass_insert(true)}
puts sprintf("%2.2fx faster than base", base.real / bench.real)
end
```
### Kết quả

``` Ruby
Testing various insert methods for 10000 inserts

ActiveRecord with transaction:
  1.29x faster than base
Raw SQL without transaction:
  5.07x faster than base
Raw SQL with transaction:
  11.46x faster than base
Single mass insert:
  70.35x faster than base
ActiveRecord::Extensions mass insert:
  2.01x faster than base
ActiveRecord::Extensions mass insert without validations:
  2.00x faster than base
```
