#!/usr/bin/rake -T

require 'simp/rake/pupmod/helpers'
require 'simp/rake/build/deps'

Simp::Rake::Beaker.new(File.dirname(__FILE__))

begin
  require 'simp/rake/build/helpers'
  BASEDIR    = File.dirname(__FILE__)
  Simp::Rake::Build::Helpers.new( BASEDIR )
rescue LoadError => e
  warn "WARNING: #{e.message}"
end

task :metadata_lint do
  sh 'metadata-json-lint --strict-dependencies --strict-license --fail-on-warnings metadata.json'
end

task :default do
  help
end

def load_modules(puppetfile)
  require 'r10k/puppetfile'

  fail "Could not find file '#{puppetfile}'" unless File.exist?(puppetfile)

  r10k = R10K::Puppetfile.new(Dir.pwd, nil, puppetfile)
  r10k.load!

  modules = {}

  r10k.modules.each do |mod|
    # Skip anything that's not pinned

    next unless mod.instance_variable_get('@args').keys.include?(:tag)

    modules[mod.name] = {
      :id          => mod.name,
      :owner       => mod.owner,
      :path        => mod.path.to_s,
      :remote      => mod.repo.instance_variable_get('@remote'),
      :desired_ref => mod.desired_ref,
      :version     => mod.desired_ref,
      :git_source  => mod.repo.repo.origin,
      :git_ref     => mod.repo.head,
      :module_dir  => mod.basedir,
      :r10k_module => mod,
      :r10k_cache  => mod.repo.repo.cache_repo
    }
  end

  return modules
end

def update_module_github_status!(mod)
  require 'json'
  require 'nokogiri'
  require 'open-uri'

  require 'highline'
  HighLine.colorize_strings

  mod[:published] ||= {}
  mod[:published]['GitHub'] ||= 'unknown'
  mod[:published]['Puppet Forge'] ||= 'unknown'

  mod_remote = mod[:remote].gsub(/\.git$/,'')

  # See if we have a valid release on GitHub
  github_releases_url = mod_remote + '/releases'

  begin
    github_query = Nokogiri::HTML(open(github_releases_url).read)

    if github_query.xpath('//a/@title').map(&:value).include?(mod[:version])
      mod[:published]['GitHub'] = 'yes'
    else
      mod[:published]['GitHub'] = 'no'
    end
  rescue
    begin
      github_releases_url = mod_remote + '/tags'
      github_query = Nokogiri::HTML(open(github_releases_url).read)

      if github_query.xpath('//a/@title').map(&:value).include?(mod[:version])
        mod[:published]['GitHub'] = 'tag'
      else
        mod[:published]['GitHub'] = 'no'
      end
    rescue
      mod[:published]['GitHub'] = 'no'
    end
  end

  if ['yes','tag'].include?(mod[:published]['GitHub'].uncolor)
    github_diff_uri = mod_remote + "/compare/#{mod[:version]}...HEAD"

    to_keep = [
      'GPGKEYS/',
      'SIMP/',
      'bin/',
      'build/',
      'data/',
      'environments/',
      'ext/',
      'files/',
      'functions/',
      'hiera.yaml',
      'lib/',
      'manifests/',
      'metadata.json',
      'rsync/',
      'sbin/',
      'scripts/',
      'share/',
      'src/',
      'tasks/',
      'templates/',
      'types/',
      'utils/'
    ]

    diff_result = Nokogiri::HTML(open(github_diff_uri).read)
    diff_items = diff_result
      .xpath("//a[contains(@href,'#diff-')]")
      .map{|x| x.text}
      .delete_if{|x|
        x.strip.empty? ||
          x.include?('→') ||
          !to_keep.find{|k| x.start_with?(k)}
      }

    if diff_items.empty?
      mod[:published]['GitHub InSync'] = 'yes'
    else
      mod[:published]['GitHub InSync'] = 'no'
      mod[:published]['GitHub Changed'] = ([''] + diff_items).sort.uniq.join("\n      * ")
    end
  end

  # See if we're a module and update the version information if necessary
  begin
    open(
      mod_remote.gsub('github.com','raw.githubusercontent.com') +
      '/' +
      mod[:desired_ref] +
      '/' +
      'metadata.json'
    ) do |fh|
      mod_info = JSON.parse(fh.read)

      mod[:version] = mod_info['version']
    end
  rescue StandardError => e
    # If we get here, we're not a module
    mod[:published]['Puppet Forge'] = 'N/A'
  end
end

def update_module_puppet_forge_status!(mod)
  require 'open-uri'

  require 'highline'
  HighLine.colorize_strings

  forge_url_base  = 'https://forgeapi.puppet.com/v3/releases/'

  mod[:published] ||= {}
  mod[:published]['Puppet Forge'] ||= 'unknown'

  unless mod[:published]['Puppet Forge'] == 'N/A'
    begin
      # Now, check the Puppet Forge for the released module
      open(URI.parse(forge_url_base + mod[:owner] + '-' + mod[:id] + '-' + mod[:version])) do |fh|
        mod[:published]['Puppet Forge'] = 'yes'
      end
    rescue StandardError => e
      mod[:published]['Puppet Forge'] = 'no'
    end
  end
end

def print_module_status(mod)
  puts "== #{mod[:owner]}-#{mod[:id]} #{mod[:version]} =="
  puts mod[:published].to_a.map{|x,y| "   * #{x} => #{y}"}.join("\n")
end
