## ruby学习部署记录

### 基础知识
#### bundle
- bundle是ruby的包管理工具
- bundle install 安装项目所需的依赖（gem包）,
第一次运行bundle install 时自动生成Gemfile.lock文件，之后每次运行bundle install 时，如果Gemfile.lock条目不变，则直接根据Gemfile.lock检查和安装gem包
- bundle常用命令：
```
bundle install `安装项目所需的gem包`
bundle update `更新gems到最新版本`
bundle exec **** `执行脚本`
bundle list | active `过滤名称包含'active'的gem包`
```

#### rvm
- 用于安装和管理Ruby环境
- 服务器上切换目前ruby版本：使用rvm管理ruby的版本，`rvm use 2.3.0 --default`切换ruby的版本
- `rvm list`查看目前安装的ruby版本
- `rvm list known`查看可安装的ruby版本
- `rvm remove 2.3.0`卸载已安装的某个版本
- rvm还可以根据项目管理不同的gemset，gemset可以理解成一个独立的虚拟gem环境，每一个gemset都是相互独立的。
- `rvm gemset create xxx`创建gemset
- `rvm use 2.3.0@xxx`,所有安装的gem包都是安装在此gemset下面
- `rvm gemset list`列出当前ruby的gemset
- `rvm gemset empty 2.3.0@xxx`清空某个gemset中的所有gem
#### Gemfile
- Gemfile总结：Gemfile是用来指定项目所需要的gems的配置文件，包括所需要的gem包及其相应的版本，并且可以指定ruby的版本
1) source指定使用的gem源（可更改为国内的源）
2) group为不同的分组，不同的环境使用的gems不同
3) 可以单独指定某个gem包路径，如从git仓库：`gem 'nokogiri', :git => 'git://github.com/tenderlove/nokogiri.git'`
4) 或者从某个文件路径：`gem "rails", :path => "vendor/rails"`
5) `bundle install --without test` `bundle install --without development test`指定不下载某个分组的gems
6) `gem 'byebug', platforms: [:mri, :mingw, :x64_mingw]`据bundle官网：If a gem should only be used in a particular platform or set of platforms, you can specify them. 
Platforms are essentially identical to groups, except that you do not need to use the --without install-time flag to exclude groups of gems for other platforms.
platforms指定gem在哪些环境下需要被使用，通常与group一起使用，这样就不需要在`bundle install`时加上`--without`
7) 可以指定gem的版本范围，也可以指定某一个固定的版本

#### rake
- rake是一门构建语言，和make类似，Rails用rake扩展来完成数据库初始化、更新等任务。
- 预编译网站用到的资源文件：`rake assets:precompile`
- rails5之前数据库等相关操作使用`rake db:migrate`,5之后可使用`rails db:migrate`

### 部署方法
- 常用的ruby server：puma,passenger,unicorn,项目中使用的是puma，puma主要基于多线程，而Unicorn基于多进程，故puma的内存占用较少
- puma的配置文件置于`config/puma.rb`
- 数据库初始化：在项目根目录下执行`bin/rails db:create RAILS_ENV=development`
- 运行：bundle exec rails s -p 3000
- 上面的部署基于development模式，如果是部署到test模式或者production模式，需修改如下几个地方：
1. bundle exec rails s -p 3000 -e production
2. 配置config/puma.rb
- 线上更新（迭代）时需要注意的地方：数据库（更新字段等）
```
#预编译网站用到的资源文件：`rake assets:precompile`(生产环境一般要做这一步)
#需要配置puma.rb文件使其为production的配置文件
rails g migration add_sex_to_users
#执行后会在db/migrate下生成一个文件，编辑这些文件即可，如果已有文件，则执行`rails db:migrate`更新数据库
#在本地执行rails s -e production时，报错说需要secret_key_base，故执行`RAILS_ENV=production bundle exec rake secret`生成key，再设置环境变量`export SECRET_KEY_BASE=xxxx`
#在config/secret.yml中引用此环境变量
production:
  secret_key_base: <%= ENV["SECRET_KEY_BASE"] %>
#再次执行rails s -e production，项目成功运行
RAILS_ENV=production rake db:migrate #生产环境数据迁移
```
- 迁移数据脚本
```
rake db:migrate #执行所有未执行的数据库迁移脚本
rake db:migrate db/migrate/20160821140233_create_articles.rb #执行指定的数据库迁移脚本
RAILS_ENV=production rake db:migrate # 设置rails运行环境为production，并执行数据库迁移脚本
rake db:drop # 丢弃当前rails环境对应的数据库实例
rake db:seed # 执行db/seeds.rb文件里的数据库初始化脚本
```

### 数据库配置
- 数据库的配置在config/database.yml中，在开发和测试环境使用sqlite即可，生产环境则可能需要使用其它的数据库来支撑。
```
# SQLite version 3.x
#   gem install sqlite3
#
#   Ensure the SQLite 3 gem is defined in your Gemfile
#   gem 'sqlite3'
#
default: &default
  adapter: sqlite3
  pool: <%= ENV.fetch("RAILS_MAX_THREADS") { 5 } %>
  timeout: 5000

development:
  <<: *default
  database: db/development.sqlite3

# Warning: The database defined as "test" will be erased and
# re-generated from your development database when you run "rake".
# Do not set this db to the same as development or production.
test:
  <<: *default
  database: db/test.sqlite3

production:
  <<: *default
  database: db/production.sqlite3
```
如上一段代码，使用的是SQLite3，若需要使用mysql或其他数据库，修改配置即可，以mysql为例，如下
```
default: &default
  adapter: mysql2
  encoding: utf8mb4
  username: root
  password: xxxxx
  host: db_server_host
  port: 3306
  
development:
  <<: *default
  database: development_db_name
  
test:
  <<: *default
  database: test_db_name
  
production:
  <<: *default
  database: production_db_name
```
注：需要在Gemfile中加入`gem 'mysql2', "~> 0.3.20"`

### redis配置
- 默认情况下，缓存仅在生产环境使用，如果本地测试环境要启动，要在config/environment/*rb文件中把`config.action.controller.perform_caching`设为true
- 
