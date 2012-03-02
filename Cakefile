fs      = require 'fs'
findit  = require './node_modules/findit'
async   = require './node_modules/async'
mint    = require 'mint'
gzip    = require 'gzip'
{exec, spawn}  = require 'child_process'
sys     = require 'util'
require 'underscore.logger'

#Tower   = require './lib/tower'

compileFile = (root, path, check) ->
  try
    data = fs.readFileSync path, 'utf-8'
    data = data.replace /require '([^']+)'\n/g, (_, _path) ->
      #_path = "#{root}/#{_path.toString().split("/")[2]}.coffee"
      _path = "#{root}/#{_path.toString().split("/")[2..-1].join("/")}.coffee"
      if !check || check(_path)
        #fs.readFileSync _path, 'utf-8'
        _root = _path.split(".")[-3..-2].join(".")
        #_root = _path.replace(/\.coffee$/, "")
        try
          compileFile(_root, _path, check) + "\n\n"
        catch error
          _console.info _path
          _console.error error.stack
          ""
      else
        ""
    data = data.replace(/module\.exports\s*=.*\s*/g, "")
    data + "\n\n"
  catch error
    ""
  
compileDirectory = (root, check, callback) ->
  code = compileFile("./src/tower/#{root}", "./src/tower/#{root}.coffee", check)
  callback(code) if callback
  code
  
compileEach = (root, check, callback) ->
  result = compileDirectory root, check, callback
  
  #fs.writeFile "./dist/tower/#{root}.coffee", result
  mint.coffee result, bare: false, (error, result) ->
    fs.writeFile "./dist/tower/#{root}.js", result
    unless error
      fs.writeFile "./dist/tower/#{root}.min.js", mint.uglifyjs(result, {})
      
obscurify = (content) ->
  replacements = {}
  replacements[process.env.NS || "Tower"] = "Tower" # use "M" for ultimate compression
  replacements["_C"]  = "ClassMethods"
  replacements["_I"]  = "InstanceMethods"
  replacements["_"]   = /Tower\.Support\.(String|Object|Number|Array|RegExp)/
  
  for replacement, lookup of replacements
    content = content.replace(lookup, replacement)
    
  content
    
task 'to-underscore', ->
  _modules  = ["string", "object", "number", "array", "regexp"]
  result    = "_.mixin\n"
  
  for _module in _modules
    content = fs.readFileSync("./src/tower/support/#{_module}.coffee", "utf-8") + "\n"
    content = content.replace(/Tower\.Support\.\w+\ *=\ */g, "")
    result += content
    
  path  = "dist/tower.support.underscore.js"  
  sizes = []
  
  result = obscurify(result)
  
  mint.coffee result, {}, (error, result) ->
    return console.log(error) if error
    
    fs.writeFileSync(path, result)
    
    sizes.push "Normal: #{fs.statSync(path).size}"
    exec "mate #{path}"
    #compressor = new Shift.UglifyJS
    #
    #compressor.render result, (error, result) ->
    #  fs.writeFileSync(path, result)
    #  
    #  sizes.push "Minfied: #{fs.statSync(path).size}"
      
      #gzip result, (error, result) ->
      #  
      #  fs.writeFileSync(path, result)
      #  
      #  sizes.push "Minified & Gzipped: #{fs.statSync(path).size}"
      #  
      #  console.log sizes.join("\n")

task 'build', ->
  result = """
window.global ||= window
module = window.module || {}
Tower = window.Tower = new (class Tower)
window.Tower.logger = if this["_console"] then _console else console

"""
  compileEach 'support', ((path) -> !!!path.match(/(dependencies)/)), (code) ->
    result += code
    
    result += fs.readFileSync("./src/tower/application/client.coffee", "utf-8") + "\n"
    
    result += fs.readFileSync("./src/tower/application/configuration.coffee", "utf-8") + "\n"
    
    compileEach 'store', ((path) -> !!path.match(/(memory|local)/)), (code) ->
      result += code
      compileEach 'client/store', null, (code) ->
        result += code
        compileEach 'model', null, (code) ->
          result += code
          compileEach 'view', null, (code) ->
            result += code
            compileEach 'controller', null, (code) ->
              result += code
              compileEach 'client/controller', null, (code) ->
                result += code
                compileEach 'client/view', null, (code) ->
                  result += code
                  compileEach 'dispatch', null, (code) ->
                    result += code
                    compileEach 'middleware', ((path) -> !!path.match(/(location|route)/)), (code) ->
                      result += code
                      
                      fs.writeFileSync('/Users/andy/badFile.coffee', result)
                      
                      # result += fs.readFileSync("./src/tower/middleware/router.coffee", "utf-8").replace(/module\.exports\s*=.*\s*/g, "") + "\n"
                      mint.coffee result, bare: false, (error, result) ->
                        _console.error error.stack if error
                        fs.writeFile "./dist/tower.js", result
                        unless error
                          #result = obscurify(result)
                
                          mint.uglifyjs result, {}, (error, result) ->
                            fs.writeFileSync("./dist/tower.min.js", result)
                
                            gzip result, (error, result) ->
                  
                              fs.writeFileSync("./dist/tower.min.js.gz", result)
                  
                              console.log "Minified & Gzipped: #{fs.statSync("./dist/tower.min.js.gz").size}"
                  
                              fs.writeFile "./dist/tower.min.js.gz", mint.uglifyjs(result, {})
            
task 'build-generic', ->
  paths   = findit.sync('./src')
  result  = ''
  
  iterate = (path, next) ->
    if path.match(/\.coffee$/) && !path.match(/(middleware|application|generator|asset|command|spec|store|path)/)
      fs.readFile path, 'utf-8', (error, data) ->
        if !data || data.match(/Bud1/)
          console.log path
        else
          data = data.replace(/module\.exports\s*=.*\s*/g, "")
          result += data + "\n"
        next()
    else
      next()

  async.forEachSeries paths, iterate, ->
    fs.writeFile './dist/tower.coffee', result
    mint.coffee result, {}, (error, result) ->
      console.log error
      fs.writeFile './dist/tower.js', result
      unless error
        fs.writeFile './dist/tower.min.js', mint.uglifyjs(result, {})
        #compressor.render result, (error, result) ->
        #  console.log error
        #  fs.writeFile './dist/tower.min.js', result

task 'clean', 'Remove built files in ./dist', ->

task 'spec', 'Run jasmine specs', ->
  spec = spawn './node_modules/jasmine-node/bin/jasmine-node', ['--coffee', './test']
  spec.stdout.on 'data', (data) ->
    data = data.toString().replace(/^\s*|\s*$/g, '')
    if data.match(/\u001b\[3\dm[\.F]\u001b\[0m/)
      sys.print data
    else
      data = "\n#{data}" if data.match(/Finished/)
      console.log data
  spec.stderr.on 'data', (data) -> console.log data.toString().trim()

task 'coffee', 'Auto compile src/**/*.coffee files into lib/**/*.js', ->
  coffee = spawn './node_modules/coffee-script/bin/coffee', ['-o', 'lib', '-w', 'src']
  coffee.stdout.on 'data', (data) -> console.log data.toString().trim()
  coffee.stderr.on 'data', (data) -> console.log data.toString().trim()
  
task 'docs', 'Build the docs', ->
  exec './node_modules/dox/bin/dox < ./lib/tower/route/dsl.js', (err, stdout, stderr) ->
    throw err if err
    console.log stdout + stderr

task 'site', 'Build site'

task 'stats', 'Build files and report on their sizes', ->
  Table = require './node_modules/cli-table'
  paths = findit.sync('./dist')
  prev  = 0
  table = new Table
    head:       ['Path', 'Size (kb)', 'Compression (%)']
    colWidths:  [50, 15, 20]
  
  for path, i in paths
    if path.match(/\.(js|coffee)$/)
      stat = fs.statSync(path)
      size = stat.size / 1000.0
      if i % 2 == 0
        percent = (size / prev) * 100.0
        percent = percent.toFixed(1)
        table.push [path, size, "#{percent} %"]
      else
        table.push [path, size, "-"]
      prev = size
      
  console.log table.toString()