---
title: Axiom Exergy Report and Query Daemon Documentation

language_tabs: # must be one of https://git.io/vQNgJ
  - json-doc
  - python

toc_footers:
  - <a href='https://sites.google.com/axiomexergy.com/wiki'>Axiom Wiki</a>
  - <a href='https://github.com/lord/slate'>Documentation Powered by Slate</a>
  - <a href='https://jsonformatter.curiousconcept.com'>JSON Formatter</a>

includes:

search: true
---

# Introduction

Welcome to the Axiom Exergy Report and Query Daemon Documentation page! Here at Axiom, we have a rich and storied history of documentation, and present this under construction documentation page as a guide to the report and query daemons on our appliance and their related config files.

Here you will find documentation of design decisions that were made in structuring the requests made to the PLCs, how the queried data is parsed and stored, and how to add new sensors, among other things.

## Structure of workflow and data

* PLCs log data (floats, ints, and bytes) in registers and coils. Each PLC has a unique IP address that can be accessed by computers on the same network.
* Query daemons are Java programs housed on the appliance. They query the PLCs and map the values of the registers to their respective keys in a JSON object. The daemons query the PLCs via MQTT, a TCP/IP message broker. The daemons publish the JSON object that is generated from a query under an MQTT publish topic on the network.
* The appliance is a Java program loaded onto the TR Raspberry Pi. It is subscribed to the MQTT topics that are published by the Query daemons. The appliance aggregates the JSON objects from the several query daemons into one large JSON object.
* The IoT report daemon takes the JSON object on the appliance, consisting of one time sample of all sensor and reference values, and uploads it to the cloud. The data is housed on InfluxDB, which is then queried by Grafana (AE Plots) and displayed.
* Commands can also be issued to the test rig via the Appliance, utilizing our cloud architecture. Changes to the command state of the CUs, TLS pump, etc. are possible. Any change made to these values will be published in a JSON object via an MQTT topic. The query daemon is subscribed to this command topic, and will write to the coils/registers as appropriate.

# The Config File

> An example, demonstrative config file is shown below. Brackets and ellipses are included to reduce the length of the document; they are not to be included in a deployed config JSON file. A complete config JSON object will be included at the bottom of the page.

```json-doc
{
	"modbus_protocol":"tcp",
	"modbus_tcp_slave_ip":"192.168.0.20",
	"mqtt_publish_topic":"tr/TRImage/TLSMain/report",
	"mqtt_publish_interval_secs":5,
	"mqtt_publish_topic":"tr/TRImage/TLSMain/command",
	"receive_float_requests":
	[
		{
			"request_start_addr":"0x70B0",
			"request_num_addrs":34,
			"register_mapping": {
				"0x70B0": {
					"json_key":"hdrLiq.T",
					"parser_type":"scalar"
				},
                [...]
			},
			"generated_measurements": {
				"Suc.SH": {
					"generator_type":"SH",
					"json_key_P":"hdrSuc.P",
					"json_key_T":"hdrSuc.T",
					"refrigerant_type":"r404a"
				},
				"Liq.SC": {
					"generator_type":"SH",
					"json_key_P":"hdrLiq.P",
					"json_key_T":"hdrLiq.T",
					"refrigerant_type":"r404a"
				},
                [...]
			}
		},
		{
			"request_start_addr":"0x7106",
			"request_num_addrs":4,
			"register_mapping": {
				"0x7106": {
					"json_key":"TLS.pump.ref",
					"parser_type":"scalar"
				},
                [...]
			}
		}
	],
	"receive_bit_requests": [
		{
			"request_start_addr":"0x431F",
			"request_num_addrs":1,
			"register_mapping": {
				"0x431F": {
					"json_key":"Abort_All_Operation",
					"parser_type":"scalar"
				}
			}
		}
	],
	"receive_byte_requests":[
	],
	"send_float_requests": [
		{
			"request_start_addr":"0x70D0",
			"request_num_addrs":2,
			"register_mapping": {
				"0x70D0": {
					"json_key":"TLS.pump.out",
					"parser_type":"scalar"
				}
			}
		}
	],
	"send_byte_requests": [
	],
	"send_bit_requests": [
		{
			"request_start_addr":"0x4320",
			"request_num_addrs":1,
			"json_key":"Receive_Abort",
			"parser_type":"boolean"
		},
         [...]
	]
}
```
### Introduction

A config file is called by a Query Daemon, and the contents of the config file determine what PLC is query'd, what registers in the PLC are scraped, and how those values are mapped. There are also MQTT broker pub and sub topics that are given in the config file.

### Some important features:

* The modbus communication protocol, tcp, is indicated in the first line. This typically will not change (unless thermistors and TM-TH8's are involved--this is currently unsupported).
* The IP address listed corresponds to the IP address of the PLC being queried. This IP address is the TR Main PLC's IP address.
* The two MQTT publish topics listed correspond to reporting data and commanding the PLC, respectively. These topics will need to be changed to correspond to the PLC.
* All of the Modbus addresses in the PLC space are given as hex addresses. Each hex address corresponds to a byte in the PLC register space, or a bit (coil) in the PLC register space.
* The rest of the JSON object is composed of `receive_float_requests`, `receive_byte_requests`, `receive_bit_requests`, `send_float_requests`, `send_byte_requests`, and `send_bit_requests`. This is where the PLC registers are mapped in the eventual JSON object.
* There are `generated_measurements` within some requests; these will generate measurements like superheat or subcool based on the temperature and pressure that are being measured in real-time from the PLC.
* All register mapping in the config JSON file has a `parser_type`, which is limited to a few supported types. Included are scalars for use with measured values, booleans for use with abort bits, and CUState for use with commanding the CUs.

### Query Workflow

1. The Java programs that query the PLCs are given a starting address in modbus space, and the number of addresses desired in the query. The daemon will execute its query by beginning at the starting address and scraping the values of every address within the query.
2. Any values that were scraped from addresses with a matching `register_mapping` address will be parsed according to the register's `parser_type` and assigned an informative JSON key string that corresponds to the real-world significance of the measurement.
3. The daemon will also evaluate any generated measurements within the same request block.

### Constraints and Best Practices

When creating the config JSON file, some hardware and communication protocol constraints must be adhered to.

* Requests have a maximum allowable register span of 60 registers; for float requests, for example, the maximum span of values that can be queried is 30. If more than 60 registers must be spanned by a certain request type, multiple request blocks will have to be included in the config JSON file.
* `generated_measurements` should be included in the same request as the values that are used for the generated measurement, after the values have been queried. This is to ensure accurate timestamping of the generated data. `generated_measurements` must also include the the `generator_type`, JSON keys for the values required in the evaluation of the quantity (for superheat, for example, temperature and pressure must be inputs), and the refrigerant that is being used.

# Handy Links

* <a href='https://jsonformatter.curiousconcept.com'>JSON Formatter</a>. Use this to verify JSON while creating or editing config files.


