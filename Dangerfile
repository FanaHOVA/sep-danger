# rubocop:disable all

# Helpers
require "json"
require "shellwords"

ISSUES_REPO = ENV.fetch('DANGER_ISSUES_REPO', 'saberespoder/inboundsms').freeze

added_lines = github.pr_diff.split("\n").select{ |line| line =~ /^\+/ }.join("\n")


# Make it more obvious that a PR is a work in progress and shouldn't be merged yet
is_wip = !!(github.pr_title =~ /\bWIP\b/i) || !!(github.pr_labels.join =~ /work in progress/i)
warn("PR is classed as Work in Progress") if is_wip

# Warn when there is a big PR
warn("Big PR") if git.lines_of_code > 500

# PR needs rebasing
fail "PR can't be merged yet, rebase needed" unless github.pr_json["mergeable"]

# Don't let testing shortcuts get into master by accident
fail "fdescribe left in tests" if `grep -r fdescribe spec/ `.length > 1
fail "fit left in tests" if `grep -r fit spec/ `.length > 1

# Mainly to encourage writing up some reasoning about the PR
fail "Please provide a summary in the Pull Request description" if github.pr_body.length < 5

# Verify that we don't test implementation details
warn "Specs are testing implementation - assigns(:something) is used"    if added_lines =~ /expect\(assigns\(:.*\)/
warn "Specs are testing implementation - respond_to(:something) is used" if added_lines =~ /expect\(.*\).to respond_to\(:.*\)/
warn "Specs are testing implementation - receive(:something) is used"    if added_lines =~ /expect\(.*\).to receive\(:.*\)/

# We don't need any debugging code in our codebase
warn "Debugging code found - puts" if added_lines =~ /^.\s*puts\b/
big_fail "Debugging code found - binding.pry" if `grep -r binding.pry lib/ app/ spec/`.length > 1
big_fail "Debugging code found - p" if added_lines =~ /^.\s*p\b/
big_fail "Debugging code found - pp" if added_lines =~ /^.\s*pp\b/
big_fail "Debugging code found - debugger" if `grep -r debugger lib/ app/ spec/`.length > 1
big_fail "Debugging code found - console.log" if `grep -r console.log lib/ app/ spec/`.length > 1
big_fail "Debugging code found - require 'debug'" if `grep -r "require \'debug\'" lib/ app/ spec/`.length > 1

# White space conventions
fail "Trailing whitespace" if added_lines =~ /\s$/
fail "Use spaces instead of tabs for indenting" if added_lines =~ /\t/

# We don't need default_scope in our codebase
if added_lines =~ /\bdefault_scope\b/
  big_fail "default_scope found. Please avoid this bad practice ([why is bad](http://stackoverflow.com/a/25087337))"
end

# Warn if 'Gemfile' was modified and 'Gemfile.lock' was not
if git.modified_files.include?("Gemfile") && !git.modified_files.include?("Gemfile.lock")
  warn("`Gemfile` was modified but `Gemfile.lock` was not")
end

if added_lines =~ /render\s.*?(&&|and)\s*return/
  big_fail "Use `return render :foo` instead of render :foo && return"
end

# Look for GIT merge conflicts
if `grep -r ">>>>\|=======\|<<<<<<<" app spec lib`.length > 1
 big_fail "Merge conflicts found"
end

# Look for timezone issues
if `grep -r "Date.today\|DateTime.now\|Time.now" app spec lib`.length > 1
  big_fail "Use explicit timezone ([See this blog](http://danilenko.org/2012/7/6/rails_timezones/))"
end

# Encourage writing specs
warn("You've added no specs for this change. Are you sure about this?") if git.modified_files.grep(/spec/).empty?

# Report failed tests
tests_failed = false
rspec_report = File.join(ENV.fetch('CIRCLE_TEST_REPORTS', '.'), 'rspec/rspec.xml')
if File.exist?(rspec_report)
  junit.parse rspec_report
  junit.report
  tests_failed = junit.failures.length > 0
  message "Rspec: #{junit.tests.length} examples, #{junit.failures.length} failures, #{junit.skipped.length} skipped"
else
  warn "junit file not found in #{rspec_report}"
end

# Setup environment for the linters (copy configs, etc)
system("
  mv package.json package.json.bak
  mv #{__dir__}/package.json .
  npm install
  mv package.json.bak package.json
  cp -v --no-clobber #{__dir__}/linter_configs/.* #{__dir__}/linter_configs/* .
")
ENV['PATH'] = "#{`npm bin`.strip}:#{ENV['PATH']}"

# Run linters
linters = `bundle exec pronto list`.split
linters_no_errors = linters.dup

linters.each do |linter|
  report = `bundle exec pronto run --runner #{linter} --commit origin/#{github.branch_for_base} -f json`
  begin
    warnings = JSON.load(report)
  rescue
    message "Linter #{linter} failed to run"
    next
  end

  linters_no_errors.delete(linter) if warnings.empty?

  warnings.each do |w|
    text = "#{linter}: #{w['message']} in `#{w['path']}:#{w['line']}`"
    if w['level'] = 'W'
      warn text
    else
      big_fail text
    end
  end
end

message "Linters #{linters_no_errors.join(', ')} reported no errors"
