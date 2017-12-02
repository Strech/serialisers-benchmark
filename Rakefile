task default: :benchmark

task :clean do
  %w(protobuf msgpack capnproto).each { |folder| rm_rf folder }
end

desc 'Benchmark protobuf msgpack capnproto json, use benchmark[capnp] to exclude it'
task :benchmark, %i(without) => %i(protobuf msgpack capnproto json) do |_, args|
  require 'benchmark/ips'
  require 'google/protobuf'
  require 'msgpack'
  require 'capn_proto'
  require 'oj'
  require_relative 'protobuf/connections_find_pb'

  GC.disable

  module Connections2
    module Find
      extend CapnProto::SchemaLoader
      load_schema('capnproto/connections_find.capnp')
    end
  end

  benchmark_config = {
    warmup: 5,
    time: 5
  }

  file_sizes = {
    protobuf: File.size(File.expand_path('protobuf/payload.bin')),
    msgpack: File.size(File.expand_path('msgpack/payload.bin')),
    capnp: File.size(File.expand_path('capnproto/payload.bin')),
    json: File.size(File.expand_path('payload.json'))
  }

  serialized_files = {
    protobuf: File.read(File.expand_path('protobuf/payload.bin')),
    msgpack: File.read(File.expand_path('msgpack/payload.bin')),
    capnp: File.read(File.expand_path('capnproto/payload.bin')),
    json: File.read(File.expand_path('payload.json'))
  }

  deserialized_files = {
    protobuf: Connections::Find::Result.decode(serialized_files[:protobuf]),
    msgpack: MessagePack.unpack(serialized_files[:msgpack]),
    capnp: Connections2::Find::Result.new_message,
    json: Oj.load(serialized_files[:json])
  }

  data = deserialized_files[:capnp].initData(deserialized_files[:json]['data'].size)
  deserialized_files[:json]['data'].each.with_index do |connection, index|
    connection_message = data[index]

    attributes = connection['attributes']
    relationships = connection['relationships']

    connection_message.id = connection['id']
    connection_message.departureTime = connection_message.initDepartureTime.tap do |timestamp|
      timestamp.seconds = Time.parse(attributes['departure_time']).to_i
    end
    connection_message.arrivalTime = connection_message.initArrivalTime.tap do |timestamp|
      timestamp.seconds = Time.parse(attributes['arrival_time']).to_i
    end
    connection_message.totalPrice = attributes['total_price']
    connection_message.currency = attributes['currency']
    connection_message.bookedOut = attributes['booked_out']

    connection_message.initMarketingCarrier.tap do |marketing_carrier|
      marketing_carrier.id = relationships['marketing_carrier']['data']['id']
    end
    connection_message.initOperatingCarrier.tap do |operating_carrier|
      operating_carrier.id = relationships['operating_carrier']['data']['id']
    end

    connection_message.initDepartureStation.tap do |station|
      station.id = relationships['departure_station']['data']['id']
    end
    connection_message.initArrivalStation.tap do |station|
      station.id = relationships['arrival_station']['data']['id']
    end
  end

  puts '                            +-------------------+'
  puts '                            |   Deserializing   |'
  Benchmark.ips do |x|
    x.config(benchmark_config)

    x.report('protobuf#decode') { Connections::Find::Result.decode(serialized_files[:protobuf]) }
    x.report('msgpack#unpack') { MessagePack.unpack(serialized_files[:msgpack]) }

    if args[:without] != 'capnp'
      x.report('capnp#from_bytes') { Connections2::Find::Result.make_from_bytes(serialized_files[:capnp]) }
    end

    x.report('oj#load') { Oj.load(serialized_files[:json]) }
    x.compare!
  end

  puts '                            +-------------------+'
  puts '                            |    Serializing    |'
  Benchmark.ips do |x|
    x.config(benchmark_config)

    x.report('protobuf#encode') { Connections::Find::Result.encode(deserialized_files[:protobuf]) }
    x.report('msgpack#pack') { MessagePack.pack(deserialized_files[:msgpack]) }

    if args[:without] != 'capnp'
      x.report('capnp#to_bytes') { deserialized_files[:capnp].to_bytes }
    end

    x.report('oj#dump') { Oj.dump(deserialized_files[:json]) }
    x.compare!
  end

  puts '                            +-------------------+'
  puts '                            |    ReadAccess     |'
  Benchmark.ips do |x|
    x.config(benchmark_config)

    x.report('protobuf#each.name') { deserialized_files[:protobuf].data.each { |c| c.departure_station.id } }
    x.report('protobuf#each[name]') { deserialized_files[:protobuf]['data'].each { |c| c['departure_station']['id'] } }

    x.report('msgpack#each[name]') { deserialized_files[:msgpack]['data'].each { |c| c['relationships']['departure_station']['data']['id'] } }

    if args[:without] != 'capnp'
      x.report('capnp#each.name') { deserialized_files[:capnp].data.each { |c| c.departureStation.id } }
      x.report('capnp#each[name]') { deserialized_files[:capnp]['data'].each { |c| c['departureStation']['id'] } }
    end

    x.report('oj#each[name]') { deserialized_files[:json]['data'].each { |c| c['relationships']['departure_station']['data']['id'] } }
    x.compare!
  end

  puts <<~EOM
                              +-------------------+
                              |       Size        |
  Comparison --------------------------------------
  EOM
  file_sizes.delete(:capnp) if args[:without] == 'capnp'
  file_sizes.sort_by { |_,v| v }.each.with_index do |(name, size), index|
    print "     #{name.to_s.rjust(9)}#size:"
    print "      #{filesize(file_sizes[name]).rjust(10)}"
    print " - #{(size.to_f / file_sizes.to_a[0][1]).round(2)}x bigger" unless index.zero?
    puts
  end
end

# == JSON

task json: %(payload.json)
file 'payload.json'

# == Msgpack
task msgpack: %w(msgpack/payload.bin)

file 'msgpack/payload.bin' do |task, _|
  require 'oj'
  require 'msgpack'

  mkdir_p 'msgpack'

  payload = Oj.load(File.read(File.expand_path('payload.json')))

  File.open(File.expand_path(task.name), 'wb') do |file|
    file.write(MessagePack.pack(payload))
  end
end

# == Protobuf
task protobuf: %w(protobuf/payload.bin)

file 'protobuf/payload.bin' => 'protobuf/connections_find_pb.rb' do |task, _|
  require 'oj'
  require 'google/protobuf'
  require_relative 'protobuf/connections_find_pb'

  payload = Oj.load(File.read(File.expand_path('payload.json')))

  data = payload['data'].each_with_object([]) do |connection, data|
    attributes = connection['attributes']
    relationships = connection['relationships']

    data << Connections::Find::Connection.new(
      id: connection['id'],
      departure_time: Google::Protobuf::Timestamp.new(
        seconds: Time.parse(attributes['departure_time']).to_i
      ),
      arrival_time: Google::Protobuf::Timestamp.new(
        seconds: Time.parse(attributes['arrival_time']).to_i
      ),
      total_price: attributes['total_price'],
      currency: attributes['currency'],
      booked_out: attributes['booked_out'],
      marketing_carrier: Connections::Find::Carrier.new(
        id: relationships['marketing_carrier']['data']['id']
      ),
      operating_carrier: Connections::Find::Carrier.new(
        id: relationships['operating_carrier']['data']['id']
      ),
      departure_station: Connections::Find::Station.new(
        id: relationships['departure_station']['data']['id']
      ),
      arrival_station: Connections::Find::Station.new(
        id: relationships['arrival_station']['data']['id']
      )
    )
  end

  File.open(File.expand_path(task.name), 'wb') do |file|
    file.write(
      Connections::Find::Result.encode(
        Connections::Find::Result.new(data: data)
      )
    )
  end
end

file 'protobuf/connections_find_pb.rb' => 'protobuf/connections_find.proto' do
  `protoc --ruby_out=. protobuf/connections_find.proto`
end

file 'protobuf/connections_find.proto' do |task, _|
  mkdir_p 'protobuf'

  File.open(task.name, 'w') do |file|
    file.write <<~EOF
    syntax = "proto3";

    package connections.find;

    import "google/protobuf/timestamp.proto";

    message Result {
        repeated Connection data = 1;
    }

    message Connection {
        string id = 1;                                // LBDI-FRPARPCD-FRPARPFO-2017-12-03T14:45-2017-12-03T13:40
        google.protobuf.Timestamp departure_time = 2; // 2017-12-03T14:45
        google.protobuf.Timestamp arrival_time = 3;   // 2017-12-03T13:40
        uint32 total_price = 4;                       // 2100
        string currency = 5;                          // EUR    TODO: Also could be replaced as enum
        bool booked_out = 6;                          // false

        Carrier marketing_carrier = 7;
        Carrier operating_carrier = 8;
        Station departure_station = 9;
        Station arrival_station = 10;

        repeated Segment segments = 11;
    }

    message Carrier {
        string id = 1; // LBDI
    }

    message Station {
        string id = 1; // FRPARPCD
    }

    message Segment {
        string id = 1; // LBDI-FRPARPCD-FRPARPFO-2017-12-03T14:45-2017-12-03T13:40
    }
    EOF
  end
end

# == Capnproto

task capnproto: %w(capnproto/payload.bin)

file 'capnproto/payload.bin' => 'capnproto/connections_find.capnp' do |task, _|
  require 'oj'
  require 'capn_proto'

  module Connections2
    module Find
      extend CapnProto::SchemaLoader
      load_schema('capnproto/connections_find.capnp')
    end
  end

  payload = Oj.load(File.read(File.expand_path('payload.json')))
  result = Connections2::Find::Result.new_message
  data = result.initData(payload['data'].size)

  payload['data'].each.with_index do |connection, index|
    connection_message = data[index]

    attributes = connection['attributes']
    relationships = connection['relationships']

    connection_message.id = connection['id']
    connection_message.departureTime = connection_message.initDepartureTime.tap do |timestamp|
      timestamp.seconds = Time.parse(attributes['departure_time']).to_i
    end
    connection_message.arrivalTime = connection_message.initArrivalTime.tap do |timestamp|
      timestamp.seconds = Time.parse(attributes['arrival_time']).to_i
    end
    connection_message.totalPrice = attributes['total_price']
    connection_message.currency = attributes['currency']
    connection_message.bookedOut = attributes['booked_out']

    connection_message.initMarketingCarrier.tap do |marketing_carrier|
      marketing_carrier.id = relationships['marketing_carrier']['data']['id']
    end
    connection_message.initOperatingCarrier.tap do |operating_carrier|
      operating_carrier.id = relationships['operating_carrier']['data']['id']
    end

    connection_message.initDepartureStation.tap do |station|
      station.id = relationships['departure_station']['data']['id']
    end
    connection_message.initArrivalStation.tap do |station|
      station.id = relationships['arrival_station']['data']['id']
    end
  end

  File.open(File.expand_path('capnproto/payload.bin'), 'wb') do |file|
    result.write(file)
  end
end

file 'capnproto/connections_find.capnp' do |task, _|
  mkdir_p 'capnproto'

  File.open(task.name, 'w') do |file|
    file.write <<~EOF
    @0x85ae6caec38e2c24;

    struct Result {
           data @0 :List(Connection);
    }

    struct Connection {
           id @0 :Text;                   # LBDI-FRPARPCD-FRPARPFO-2017-12-03T14:45-2017-12-03T13:40
           departureTime @1 :Timestamp;   # 2017-12-03T14:45
           arrivalTime @2 :Timestamp;     # 2017-12-03T13:40
           totalPrice @3 :UInt32;         # 2100
           currency @4 :Text;             # EUR    TODO: Also could be replaced as enum
           bookedOut @5 :Bool;            # false

           marketingCarrier @6 :Carrier;
           operatingCarrier @7 :Carrier;
           departureStation @8 :Station;
           arrivalStation @9 :Station;

           segments @10 :List(Segment);
     }

     struct Carrier {
            id @0 :Text; # LBDI
     }

     struct Station {
            id @0 :Text; # FRPARPCD
     }

     struct Segment {
            id @0 :Text; # LBDI-FRPARPCD-FRPARPFO-2017-12-03T14:45-2017-12-03T13:40
     }

     struct Timestamp {
            seconds @0 :Int32;
     }
    EOF
  end
end

# == Utils
def filesize(size)
  units = %w(B KiB MiB GiB TiB Pib EiB)

  return '0.0 B' if size.zero?
  exp = (Math.log(size) / Math.log(1024)).to_i
  exp = 6 if exp > 6

  format('%.1f %s', size.to_f / 1024**exp, units[exp])
end
