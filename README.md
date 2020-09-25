import argparse
import xml.etree.ElementTree as ET
import datetime

parser = argparse.ArgumentParser(description='Read RTDM SOAP log to parse actual request timeout.')
parser.add_argument('infile', metavar='Infile', type=str, help='Log to process')
parser.add_argument('okfile', metavar='OK_file', type=str, help='Result file for OK results')
parser.add_argument('badfile', metavar='Other_file', type=str, help='Result file for other results')
args = parser.parse_args()

infile = vars(args)['infile']; print("Infile: %s" %infile)
okfile = vars(args)['okfile']; print("OK Outfile: %s" %okfile)
badfile = vars(args)['badfile']; print("Other Outfile: %s" %badfile)

xpath_start_time = ".//ns2:EventResponse/ns2:Header/ns2:StartTime"
xpath_end_time = ".//ns2:EventResponse/ns2:Header/ns2:CompletionTime"
xpath_result = ".//ns2:Data[@name='RESULT']/ns2:String/ns2:Val"
xpath_entity_id = ".//rtdm:Data[@name='ID_CUID']/rtdm:String/rtdm:Val"


f = open(infile, 'r')
okfile = open(okfile, 'w')
badfile = open(badfile, 'w')

for line in f.readlines():
    id_cuid = None
    #print('='*10)
    date = line.split('|')[0]
    req_type = line.split('|')[1]

    #print("date: %s; req_type: %s" %(date, req_type))
    if req_type == 'sent':
        out_xml = line.split('] for request [')[0]
        in_xml = line.split('] for request [')[1]

        out_xml = out_xml[out_xml.find('[')+1:]
        in_xml = in_xml[:in_xml.find(']')]

        #print("In xml: %s" %in_xml)
        #print("\nOut xml: %s" %out_xml)

        in_xml = ET.fromstring(in_xml)

        out_xml = ET.fromstring(out_xml)

        in_prefix_map = {
                "soapenv": "http://schemas.xmlsoap.org/soap/envelope/",
                "rtdm":"http://www.sas.com/xml/schema/sas-svcs/rtdm-1.1"
        }
        id_cuid = in_xml.find(xpath_entity_id, in_prefix_map).text

        out_prefix_map = {
                "SOAP-ENV": "http://schemas.xmlsoap.org/soap/envelope/",
                "ns2":"http://www.sas.com/xml/schema/sas-svcs/rtdm-1.1"
        }
        # no time for fancy timezones
        start_time = out_xml.find(xpath_start_time,out_prefix_map).text.split('+')[0]
        end_time = out_xml.find(xpath_end_time, out_prefix_map).text.split('+')[0]
        result = out_xml.find(xpath_result, out_prefix_map).text


        start_time = datetime.datetime.strptime(start_time, '%Y-%m-%dT%H:%M:%S.%f')
        end_time = datetime.datetime.strptime(end_time, '%Y-%m-%dT%H:%M:%S.%f')
        diff = end_time - start_time

        #print("\nGot id_cuid: %s" %id_cuid)
        #print("Got start_time: %s" %start_time)
        #print("Got end_time: %s" %end_time)
        #print("Got result: %s" %result)
        #print("Date diff: %s" %diff)

        if result == "OK":
            okfile.write("%s\t%s\t%s\t%s\n" %(diff.__str__(), start_time, str(id_cuid), result))
        else:
            badfile.write("%s\t%s\t%s\t%s\n" %(diff.__str__(), start_time, str(id_cuid), result))


f.close()
okfile.close()
badfile.close()
