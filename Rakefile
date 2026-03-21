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

# Require confirmation before clobber deletes the transcription cache
task :clobber do
  print "This will delete the OCR/transcription cache (cache/), which may take hours to rebuild.\nType CLOBBER to confirm: "
  $stdout.flush
  abort "Aborted." unless $stdin.gets.to_s.strip == 'CLOBBER'
end

desc "Step 1: Extract from MBOX"
task :extract do
  puts "--- [1] Extracting from MBOX ---"
  system "scripts/import_mbox"
end

desc "Interactively review suspect attachments (run after rake extract)"
task :detect_excluded_attachments do
  puts "--- [1.x] Detecting junk attachments (interactive) ---"
  system "scripts/detect_excluded_attachments"
end

desc "Step 1.1: Delete excluded attachments by MD5"
task :delete_excluded_attachments => :extract do
  puts "--- [1.1] Deleting excluded attachments ---"
  system "scripts/delete_excluded_attachments"
end

desc "Step 1.2: Remove duplicate attachments"
task :deduplicate_attachments => :delete_excluded_attachments do
  puts "--- [1.2] Deduplicating Attachments ---"
  system "scripts/deduplicate_attachments -d"
end

desc "Step 2: Compile email narrative (depends on dedupe)"
task :compile_emails => :deduplicate_attachments do
  puts "--- [2] Compiling Email Narrative ---"
  system "scripts/compile_emails"
end

desc "Step 3: Compile/OCR attachments (depends on dedupe)"
task :compile_docs => :deduplicate_attachments do
  puts "--- [3] Compiling Attachment Text (with OCR) ---"
  system "scripts/transcribe_attachments"
end

desc "Step 4: Sync to Context for Gemini"
task :sync => [:compile_emails, :compile_docs] do
  puts "--- [4] Syncing to Context ---"
  system "scripts/create_context"
end

task :default => :sync
