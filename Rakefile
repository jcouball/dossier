require 'fileutils'
require 'rake/clean'

BASE_DIR = File.expand_path('~/dossier')
CONTEXT_DIR = File.join(BASE_DIR, 'context')
WORK_DIR = File.join(BASE_DIR, 'work')

# `rake clean`   — removes all generated intermediate and output files
CLEAN.include('work/')
CLEAN.include('context/')

# `rake clobber` — everything in CLEAN plus the OCR cache
CLOBBER.include('cache/')

# `rake clobber` — everything in CLEAN plus build artifacts
CLOBBER.include('Gemfile.lock')

desc "Step 1: Extract from MBOX"
task :extract do
  puts "--- [1] Extracting from MBOX ---"
  system "scripts/import_mbox"
end

desc "Step 1.1: Remove known junk files by MD5"
task :filter_md5 => :extract do
  puts "--- [1.1] Filtering known junk (MD5) ---"
  system "scripts/delete_by_md5"
end

desc "Step 1.2: Remove duplicate attachments"
task :dedupe => :filter_md5 do
  puts "--- [1.2] Deduplicating Attachments ---"
  system "scripts/deduplicate -d"
end

desc "Step 2: Compile email narrative (depends on dedupe)"
task :compile_emails => :dedupe do
  puts "--- [2] Compiling Email Narrative ---"
  system "scripts/compile_emails"
end

desc "Step 3: Compile/OCR attachments (depends on dedupe)"
task :compile_docs => :dedupe do
  puts "--- [3] Compiling Attachment Text (with OCR) ---"
  system "scripts/transcribe_attachments"
end

desc "Step 4: Sync to Context for Gemini"
task :sync => [:compile_emails, :compile_docs] do
  puts "--- [4] Syncing to Context ---"
  system "scripts/create_context"
end

task :default => :sync
