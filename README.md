# Installation

## Mac

```bash
$ brew install protobuf
$ brew install capnp
$ bundle -j8
```

## Linux

```bash
$ open https://github.com/google/protobuf/blob/master/src/README.md
$ open https://github.com/google/protobuf/releases
$ open https://github.com/cstrahan/capnp-ruby/pull/6
$ bundle -j8
```

### Capnproto gem

```bash
$ open https://github.com/cstrahan/capnp-ruby/pull/6
$ CXX=clang++ CXXFLAGS=gnu++11 gem install capn_proto --pre
```

or

```bash
$ git clone git@github.com:cstrahan/capnp-ruby.git
$ cd capnp-ruby
$ vi ext/capn_proto/extconf.rb
```

```diff
diff --git a/ext/capn_proto/extconf.rb b/ext/capn_proto/extconf.rb
index 3f35273..8b19ced 100644
--- a/ext/capn_proto/extconf.rb
+++ b/ext/capn_proto/extconf.rb
@@ -20,8 +20,16 @@ else
   $CPPFLAGS += " -DNDEBUG"
 end

+CONFIG['optflags'] += ' -std=gnu++11' # Should be picked by ENV setting
+
 $LDFLAGS += " -lcapnpc"
 $LDFLAGS += " -lcapnp"
 $LDFLAGS += " -lkj"
```

```bash
$ gem build capn_proto.gemspec
$ CXX=clang++ gem install capn_proto-0.0.1.alpha.8.gem
```

# Benchmarking

```bash
$ bundle exec rake
$ bundle exec rake benchmark[capnp] # run without capnp
```

# JSON

Initial data is represented like this

```json
{
  "data": [
    {
      "id": "LBDI-FRPARPCD-FRPARPFO-2017-12-03T14:45-2017-12-03T13:40",
      "type": "connections",
      "attributes": {
        "departure_time": "2017-12-03T14:45",
        "arrival_time": "2017-12-03T13:40",
        "total_price": 2100,
        "currency": "EUR",
        "booked_out": true
      },
      "relationships": {
        "marketing_carrier": {
          "data": {
            "id": "LBDI",
            "type": "marketing_carriers"
          }
        },
        "departure_station": {
          "data": {
            "id": "FRPARPCD",
            "type": "stations"
          }
        },
        "arrival_station": {
          "data": {
            "id": "FRPARPFO",
            "type": "stations"
          }
        },
        "operating_carrier": {
          "data": {
            "id": "LBDI",
            "type": "operating_carriers"
          }
        },
        "segments": {
          "data": [
            {
              "id": "LBDI-FRPARPCD-FRPARPFO-2017-12-03T14:45-2017-12-03T13:40",
              "type": "segments"
            }
          ]
        }
      }
    }
  ]
}
```
