---
layout: post
title: 在 Rails 中集成 DataTable
tags: rails
---

> 之前写过一篇关于如何在 Laravel 中的 DataTable 添加一个导出按钮的文章《[在 Laravel 上实现 DataTable 的 Excel 导出按钮]({% post_url 2018/2018-01-25-datatable-export-button %})》，其实这篇文章也讲解了 DataTable 在 Laravel 的使用和配置。今天再介绍一下如何在 Rails 中使用 DataTable。

### 安装 Gem 包

Rails 上的 DataTable 会用到两个 Gem 包，分别是：

* 前端：[https://github.com/mkhairi/jquery-datatables](https://github.com/mkhairi/jquery-datatables)
* 后端：[https://github.com/jbox-web/ajax-datatables-rails](https://github.com/jbox-web/ajax-datatables-rails)

打开 Gemfile 添加:

```
gem 'jquery-datatables'
gem 'ajax-datatables-rails'
```

在继续下一步之前，请先检查一下你是否安装了 `jquery-rails`，因为他是 DataTable 的基础依赖，而在 Rails 5 之后已经不默认安装了。

然后运行 `bundle install` 完成安装。

### 配置前端样式

首先我们来看前端的配置。

```
# 集成 bootstrap4 的样式，如果你不用的话去掉 bootstrap4 即可
$ rails g jquery:datatables:install bootstrap4
```

这条命令会自动在 app/assets 中配置 js/css 的依赖。需要注意的是，如果你使用的是 scss 的格式而不是默认的 css，则要在 `app/assets/stylesheets/application.scss` 中加入 `@import "datatables";`，然后新建文件 `app/assets/stylesheets/datatables.scss` 并添加如下内容：

```
@import 'datatables/dataTables.bootstrap4';
@import 'datatables/extensions/Responsive/responsive.bootstrap4';
@import 'datatables/extensions/Buttons/buttons.bootstrap4';
```

这样 DataTable 的前端就配置完成了。

### 配置服务端

1. 创建配置文件

    ```
    $ bundle exec rails generate datatable:config
    ```

    这条命令会创建 `config/initializers/ajax_datatables_rails.rb` 文件，在其中配置你自己的数据库适配器，比如笔者使用的 postgresql：

    ```
    config.db_adapter = :pg
    ```

2. 生成 datatable 类

    ```
    $ rails g datatable User
    ```

    这条命令会创建 `app/datatables/user_datatable.rb` 文件。

3. 创建 view

    ```
    <table id="users-datatable" data-source="<%= users_path(format: :json) %>">
      <thead>
        <tr>
          <th>ID</th>
          <th>Company</th>
          <th>Name</th>
          <th>Email</th>
          <th>Action</th>
        </tr>
      </thead>
      <tbody>
      </tbody>
    </table>
    ```

    DataTable 会自动填充 tbody 内容，注意这里用 json 格式请求 UserController#index 路由。

4. 配置 UserDatatable 类

    * 申明字段映射

    ```
    def view_columns
      @view_columns ||= {
        id:           { source: "User.id" },
        company_name: { source: "Company.name", searchable: false, orderable: false },
        name:         { source: "User.name" },
        email:        { source: "User.email" },
        action:       { source: "custom_action", orderable: false, searchable: false }
      }
    end
    ```

    这里的 `company_name` 读取的是 `Company` 类的 `name` 字段，笔者暂时没解决如何在关联字段上搜索与排序，所以 `serchable` 和 `orderable` 都是 `false`。最后一个字段 `action` 留作自定义，之后我们会填入相应内容。

    * 填充字段内容

    ```
    def data
      records.map do |record|
        {
          id:           record.id,
          company_name: record.company.name,
          name:         record.name,
          email:        record.email,
          action:       record.is_ok ? "<i class='fas fa-check text-success'></i>".html_safe : "<a href='/url/#{record.id}' class='btn btn-primary'>按钮</a>".html_safe,
        }
      end
    end
    ```

    `record` 就是遍历的 `user` 对象，所以直接对其操作取值就行了，所以可以直接用 `record.company.name`。最后的 `action` 是自己拼接的 html，用了三元判断符，为真则表示成 `fontawesome` 的一个勾符号，为假则表示成一个按钮。

    * 获取 `records`

    ```
    def get_raw_records
      User.inlcudes(:company)
    end
    ```

    因为这里用到了关联，所以为了避免 N+1 使用了 `includes`, 如果没有关联直接用 `User.all` 即可，`DataTable` 会自动加上默认的 `limit(10)`，当然这个分页数可以自定义设置（页面点击选择，或者初始化时更改默认值）。

    * 编写 controller 的 index 方法

    ```
    def index
      respond_to do |format|
        format.html
        format.json { render json: UserDatatable.new(view_context) }
      end
    end
    ```

    第一次打开 `index` 时候会返回 html 并打开页面，然后通过页面的 js 启动 ajax 请求获取 `DataTable` 表格数据以 json 格式返回。

    ***注:*** 官网上的写法是 `format.json { render json: UserDatatable.new(params) }`，而笔者测试下来正确的写法应该是 `view_context`）
    ***再注:*** 2019/07/07，后来又更新了版本，`view_context` 弃用了，`params` 是现在正确的写法，见[详情](https://github.com/jbox-web/ajax-datatables-rails#warnings)

    * 编写 index 页面的 js

    在 Rails 你可以用原生的 coffee 写也可以选择 js。笔者方便起见直接写在了 `index.html.erb` 文件的底部。

    ```
    <script>
      $(document).ready(function() {
        $("#users-datatable").dataTable({
          "language": {
            "url": "/localization/dataTable/Chinese.json"
          },
          "processing": true,
          "serverSide": true,
          "ajax": $("#users-datatable").data("source"),
          "pagingType": "full_numbers",
          "columns": [
            { "data": "id" },
            { "data": "company_name" },
            { "data": "name" },
            { "data": "email" },
            { "data": "action" },
          ]
        });
      });
    </script>
    ```

    这里笔者使用 `language` 参数加上了中文的翻译，这个写法的优点再与简洁并且代码分离，把翻译的内容单独放在 `config/locales` 文件夹下，保持结构一致性；但是缺点是它会多发起一次 `request` 去获取后台的翻译文件。我们先来看看这个写法的逻辑。

    这里的 `url` 参数对应的 `route` 如下：

    ```
    get '/localization/dataTable/Chinese.json', to: "home#dataTable_cn"
    ```

    再来看 HomeController#dataTable_cn 方法：

    ```
    def dataTable_cn
      render file: "/app/config/locales/dataTable.zh-CN.json"
    end
    ```

    最终它读取了 `/app/config/locales/dataTable.zh-CN.json` 文件：

    ```
    {
      "sProcessing":   "处理中...",
      "sLengthMenu":   "显示 _MENU_ 项结果",
      "sZeroRecords":  "没有匹配结果",
      "sInfo":         "显示第 _START_ 至 _END_ 项结果，共 _TOTAL_ 项",
      "sInfoEmpty":    "显示第 0 至 0 项结果，共 0 项",
      "sInfoFiltered": "(由 _MAX_ 项结果过滤)",
      "sInfoPostFix":  "",
      "sSearch":       "搜索:",
      "sUrl":          "",
      "sEmptyTable":     "表中数据为空",
      "sLoadingRecords": "载入中...",
      "sInfoThousands":  ",",
      "oPaginate": {
          "sFirst":    "首页",
          "sPrevious": "上页",
          "sNext":     "下页",
          "sLast":     "末页"
      },
      "oAria": {
          "sSortAscending":  ": 以升序排列此列",
          "sSortDescending": ": 以降序排列此列"
      }
    }
    ```

    如果不想为了翻译而多发起一次请求的话，另一种方法就是直接把这些翻译内容写进 js，缺点也很明显，就是代码很乱结构不完整。

### DataTable 与 Turbolinks 的冲突

启用 DataTable 后，笔者发现一个比较严重的问题：使用浏览器自带的后退功能回到 `index` 页面时，`DataTable` 的操作表盘会重复渲染一次，如下图示：

![datatable-twice](/assets/img/posts/2018/rails-datatable/datatable-twice.jpeg "DataTable Refreshed Twice")

如果反复回退则操作表盘会越来越多，页面相当变态... 而且即使是刷新页面，多个表盘也会先闪现一下再回复正常。一开始以为是 `DataTable` 的设置出了问题，最后怀疑也许是缓存机制的问题，进而查到了 turbolinks，[这篇文章](https://yafeilee.me/blogs/88)最终解释了原因。

其中，“典型问题三: body 中更新某个 DOM 节点会导致页面缓存重复的问题” 就讲解了几乎一模一样的问题，笔者采用的具体办法就是添加 `<meta name="turbolinks-cache-control" content="no-cache">` 来关闭当前页面的缓存。

回过头来说，笔者认为 turbolinks 是一个非常强大的 SPA（Single Page App，单页应用）工具，它远比 `AngularJS` 等第三方框架轻量，甚至很多地方强于他们。虽然很多人都会习惯性默认关闭 turbolinks，但笔者建议如果你不了解 turbolinks 的话不要轻易人云亦云，turbolinks 的强大会在你慢慢了解它后震撼到你。

---

## 参考资料：

* [jquery-datatables](https://github.com/mkhairi/jquery-datatables)
* [ajax-datatables-rails](https://github.com/jbox-web/ajax-datatables-rails)
* [Bug working with turbolinks](https://yafeilee.me/blogs/88)