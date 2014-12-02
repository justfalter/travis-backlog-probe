require 'date'
require 'net/http'
require 'uri'
require 'openssl'

WEBHOOK_URL = 'https://webhooks.gitter.im/e/0a8a11aa7e743be8ca92'
PROBE_FILENAME = "PROBE_NUMBER"
TIMESTAMP_FILE = "timestamp.txt"

def get_time_from_ntp
  before = DateTime.now
  output = `ntpdate -q pool.ntp.org`
  delta = DateTime.now - before
  unless $?.success?
    raise "ntpdate exited with #{$?.exitstatus}"
  end
  if match = output.match(/^(.*) ntpdate\[.*adjust time server/)
    time = DateTime.parse(match[1])
    # adjust for the time it took to run ntpdate
    time = time - delta
    return time
  else
    raise "Failed to parse ntpdate output! : #{output}"
  end
end

def get_time_from_file(filename)
  time_s = File.read(filename)
  return DateTime.rfc3339(time_s)
end

def write_time_to_file(time, filename)
  File.open(filename, 'w') do |fio|
    fio.write(time.rfc3339)
  end
end

def send_message(message)
  url = URI(WEBHOOK_URL)
  params = { message: message }
  encoded_params = URI.encode_www_form(params)
  Net::HTTP.start(url.host, url.port, :use_ssl => url.scheme == 'https') do |http|
    http.verify_mode = OpenSSL::SSL::VERIFY_NONE


    require 'pp'
    pp [url.request_uri]
    pp http
    http.post(url.request_uri, encoded_params)
  end
end

def report_start_delay(seconds)
  job_url = "https://travis-ci.org/justfalter/travis-backlog-probe/builds/#{ENV['TRAVIS_JOB_ID']}"
  job = ENV['TRAVIS_JOB_NUMBER']
  commit = ENV['TRAVIS_COMMIT']
  send_message("**[Job #{job}](#{job_url})**: #{seconds} seconds")
end

desc "Record the time to #{TIMESTAMP_FILE}"
task :record_time do
  write_time_to_file(get_time_from_ntp, TIMESTAMP_FILE)
end

desc "Get's the NTP time"
task :get_ntp_time do
  puts get_time_from_ntp()
end

desc "Reads the timestamp time"
task :read_timestamp do
  puts get_time_from_file(TIMESTAMP_FILE)
end

task :report_start_delay do
  time = get_time_from_ntp()
  time2 =  get_time_from_file(TIMESTAMP_FILE)
  seconds = (time.to_time - time2.to_time)
  report_start_delay(seconds)
end

def do_command(cmd)
  system(cmd) or raise("Command failed: #{cmd}")
end

task :send_probe => [ :record_time ] do
  do_command("git add #{TIMESTAMP_FILE}")
  do_command("git commit -m 'sending probe!'")
  do_command("git push origin master")
end
