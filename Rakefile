require 'open-uri'
require 'nokogiri'
require 'yaml'

namespace :tlds do
  desc 'Grab tlds from iana and save to tld_lib.yml'
  task :iana_update do
    doc = Nokogiri::HTML(open('http://www.iana.org/domains/root/db'))
    tlds = []
    types = {
      'country' => /country-code/,
      'generic' => /generic|sponsored|infrastructure|generic-restricted/,
    }

    doc.css('table#tld-table tr').each do |tr|
      info = tr.css('td')
      unless info.empty?
        tlds << parse_node(info)
      end
    end

    yml = {}
    types.each do |name, regex|
      yml[name] = select_tld(tlds, regex)
    end

    yml["generic"] << "onion"

    File.open(repo_path('tld_lib.yml'), 'w') do |file|
      file.write(yml.to_yaml)
    end

    File.open(repo_path("TldLists.java"), 'w') do |file|
      file.write(<<-EOF
// Auto-generated by conformance/Rakefile

package com.twitter;

import java.util.Arrays;
import java.util.List;

public class TldLists {
  public static final List<String> GTLDS = Arrays.asList(
#{yml["generic"].map {|el| "    \"#{el}\""}.join(",\n")}
  );

  public static final List<String> CTLDS = Arrays.asList(
#{yml["country"].map {|el| "    \"#{el}\""}.join(",\n")}
  );
}
EOF
      )
    end
  end

  desc 'Update tests from tld_lib.yml'
  task :generate_tests do
    test_yml = { 'tests' => { } }

    path = repo_path('tld_lib.yml')
    yml = YAML.load_file(path)
    yml.each do |type, tlds|
      test_yml['tests'][type] = []
      tlds.each do |tld|
        test_yml['tests'][type].push(
          'description' => "#{tld} is a valid #{type} tld",
          'text' => "https://twitter.#{tld}",
          'expected' => ["https://twitter.#{tld}"],
        )
      end
    end

    File.open('tlds.yml', 'w') do |file|
      file.write(test_yml.to_yaml)
    end
  end
end

def parse_node(node)
  {
    domain: node[0].text.gsub(/[\.\s]+/, '').gsub("\u200f", '').gsub("\u200e", ""),
    type: node[1].text
  }
end

def select_tld(tlds, type)
  # Reverse tlds to make sure tld regex can match longer one when subset exists
  tlds.select {|i| i[:type] =~ type}.map {|i| i[:domain]}.sort.reverse
end

def repo_path(*path)
  File.join(File.dirname(__FILE__), *path)
end
