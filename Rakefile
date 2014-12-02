require 'date'
require 'net/http'
require 'uri'
require 'openssl'

WEBHOOK_URL = 'https://webhooks.gitter.im/e/0a8a11aa7e743be8ca92'
PROBE_FILENAME = "PROBE_NUMBER"
TIMESTAMP_FILE = "timestamp.txt"
NTP_PACKET_SIZE= 48
EPOCH_SECONDS = 2208988800

def get_time_from_ntp
  start = Time.now
  require 'socket'
  s = UDPSocket.new
  ntppacket = [0b11100011, 0, 6, 0xEC, 0, 49, 0x4E, 49, 52,0,0,0,0].pack("C4QC4Q4")
  3.times do
    s.send ntppacket, 0, "pool.ntp.org", 123
  end
  response = s.recv(NTP_PACKET_SIZE)
  delta = Time.now - start
  z, time = response.unpack("A40N")
  epoch = time - EPOCH_SECONDS - delta
  time =  Time.at(epoch)
  return time.to_datetime
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
  current_time = get_time_from_ntp()
  timestamp =  get_time_from_file(TIMESTAMP_FILE)
  puts "Timestamp: #{timestamp.to_s}"
  puts "Current time: #{current_time.to_s}"
  seconds = (current_time.to_time - timestamp.to_time)
  puts "Delta: #{seconds}"
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
