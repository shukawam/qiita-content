---
title: 【小ネタ】 OKE のクイック作成で作成される VCN の関連リソースを一括削除する
tags:
  - oci
  - oraclecloud
  - OKE
private: false
updated_at: '2023-01-17T20:12:20+09:00'
id: 21d42a6f39f184102fcf
organization_url_name: oracle
slide: false
ignorePublish: false
---
# はじめに

OKE(Oracle Container Engine for Kubernetes) のクイック作成に限らず、現在 VCN は関連リソースまで含めたカスケード削除ができないので、検証後のゴミ掃除がちょっと面倒なわけです。それを解決するべく、Python でスクリプトを書いてみました。
※実行すると、問答無用で関連リソースが削除されるので、ご使用は各自の判断にお任せします...

ソースコード: [https://github.com/shukawam/oci-scripts/blob/main/script/delete-oke-quickstart-vcn.py](https://github.com/shukawam/oci-scripts/blob/main/script/delete-oke-quickstart-vcn.py)

# スクリプトと実行方法

```delete-oke-quickstart-vcn.py
import argparse
import sys

import oci
from oci.core import VirtualNetworkClient
from oci.core.models import UpdateRouteTableDetails
from oci.core.models import UpdateSecurityListDetails


class CustomHelpFormatter(argparse.HelpFormatter):
    def __init__(self, prog, indent_increment=2, max_help_position=50, width=None):
        super().__init__(prog, indent_increment, max_help_position, width)


def parseInput():
    parser = argparse.ArgumentParser(
        description="OKE quick-start VCN automatically deleting script.", formatter_class=CustomHelpFormatter)
    parser.add_argument('--compartment-id', type=str, help="[required]")
    parser.add_argument('--vcn-id', type=str, help="[required]")
    parser.add_argument('--profile', type=str,
                        default="DEFAULT", help="[optional]")
    if len(sys.argv) < 2:
        parser.print_help()
        sys.exit(1)
    else:
        return parser.parse_args()


def delete_route_tables_rules(quick_start_vcn_ocid: str, vcn_client: VirtualNetworkClient, compartment_id: str):
    print("*** START: DELETE ROUTE TABLE RULES. ***")
    route_tables = vcn_client.list_route_tables(
        compartment_id=compartment_id).data
    quick_start_route_tables = list(
        filter(lambda d: d.vcn_id == quick_start_vcn_ocid, route_tables))
    for route_table in quick_start_route_tables:
        vcn_client.update_route_table(
            rt_id=route_table.id,
            update_route_table_details=UpdateRouteTableDetails(
                route_rules=[]
            )
        )
    print("*** END: DELETE ROUTE TABLE RULES. ***")


def delete_gateway_resouces(quick_start_vcn_ocid: str, vcn_client: VirtualNetworkClient, compartment_id: str):
    print("*** START: DELETE GATEWAY RESOURCES. ***")
    nat_gateway_id = list(
        filter(lambda d: d.vcn_id == quick_start_vcn_ocid,
               vcn_client.list_nat_gateways(compartment_id=compartment_id).data)
    )[0].id
    print("*** DELETING NAT GATEWAY... ***")
    vcn_client.delete_nat_gateway(nat_gateway_id=nat_gateway_id)
    svc_gateway_id = list(
        filter(lambda d: d.vcn_id == quick_start_vcn_ocid,
               vcn_client.list_service_gateways(compartment_id=compartment_id).data)
    )[0].id
    print("*** DELETING SERVICE GATEWAY... ***")
    vcn_client.delete_service_gateway(service_gateway_id=svc_gateway_id)
    internet_gateway_id = list(
        filter(lambda d: d.vcn_id == quick_start_vcn_ocid,
               vcn_client.list_internet_gateways(compartment_id=compartment_id).data)
    )[0].id
    print("*** DELETING INTERNET GATEWAY... ***")
    vcn_client.delete_internet_gateway(ig_id=internet_gateway_id)
    print("*** END: DELETE GATEWAY RESOURCES. ***")


def delete_security_list_rules(quick_start_vcn_ocid: str, vcn_client: VirtualNetworkClient, compartment_id: str):
    print("*** START: DELETE SECURITY LIST RULES. ***")
    security_lists = vcn_client.list_security_lists(
        compartment_id=compartment_id).data
    quick_start_security_lists = list(
        filter(lambda d: d.vcn_id == quick_start_vcn_ocid, security_lists))
    for security_list in quick_start_security_lists:
        vcn_client.update_security_list(
            security_list_id=security_list.id,
            update_security_list_details=UpdateSecurityListDetails(
                egress_security_rules=[],
                ingress_security_rules=[]
            )
        )
    print("*** END: DELETE SECURITY LIST RULES. ***")


def delete_subnets(quick_start_vcn_ocid: str, vcn_client: VirtualNetworkClient, compartment_id: str):
    print("*** START: DELETE SUBNET. ***")
    subnet_lists = vcn_client.list_subnets(compartment_id=compartment_id).data
    quick_start_subnet_list = list(
        filter(lambda d: d.vcn_id == quick_start_vcn_ocid, subnet_lists))
    for subnet_list in quick_start_subnet_list:
        vcn_client.delete_subnet(subnet_id=subnet_list.id)
    print("*** END: SECURITY SUBNET. ***")


def delete_security_list(quick_start_vcn_ocid: str, vcn_client: VirtualNetworkClient, compartment_id: str):
    print("*** START: DELETE SECURITY LIST. ***")
    security_lists = vcn_client.list_security_lists(
        compartment_id=compartment_id).data
    quick_start_vcn_ocid.startswith
    quick_start_security_list = list(
        filter(lambda d: d.vcn_id == quick_start_vcn_ocid and d.display_name.startswith('oke-svclbseclist') == False, security_lists))
    for security_list in quick_start_security_list:
        vcn_client.delete_security_list(security_list_id=security_list.id)
    print("*** END: DELETE SECURITY LIST. ***")


def delete_route_table(quick_start_vcn_ocid: str, vcn_client: VirtualNetworkClient, compartment_id: str):
    print("*** START: DELETE ROUTE TABLE. ***")
    route_table_lists = vcn_client.list_route_tables(
        compartment_id=compartment_id).data
    quick_start_route_table = list(
        filter(lambda d: d.vcn_id == quick_start_vcn_ocid and d.display_name.startswith('oke-public-routetable') == False, route_table_lists))
    for route_table in quick_start_route_table:
        vcn_client.delete_route_table(rt_id=route_table.id)
    print("*** END: DELETE ROUTE TABLE. ***")


def delete_vcn(quick_start_vcn_ocid: str, vcn_client: VirtualNetworkClient):
    print("*** START: DELETE VCN. ***")
    vcn_client.delete_vcn(vcn_id=quick_start_vcn_ocid)
    print("*** END: DELETE VCN. ***")


def main():
    args = parseInput()
    config = oci.config.from_file("~/.oci/config", args.profile)
    compartment_id = args.compartment_id
    quick_start_vcn_ocid = args.vcn_id
    vcn_client = VirtualNetworkClient(config)
    delete_route_tables_rules(
        quick_start_vcn_ocid=quick_start_vcn_ocid, vcn_client=vcn_client, compartment_id=compartment_id)
    delete_gateway_resouces(
        quick_start_vcn_ocid=quick_start_vcn_ocid, vcn_client=vcn_client, compartment_id=compartment_id)
    delete_security_list_rules(
        quick_start_vcn_ocid=quick_start_vcn_ocid, vcn_client=vcn_client, compartment_id=compartment_id)
    delete_subnets(
        quick_start_vcn_ocid=quick_start_vcn_ocid, vcn_client=vcn_client, compartment_id=compartment_id)
    delete_security_list(
        quick_start_vcn_ocid=quick_start_vcn_ocid, vcn_client=vcn_client, compartment_id=compartment_id)

    delete_route_table(quick_start_vcn_ocid=quick_start_vcn_ocid,
                       vcn_client=vcn_client, compartment_id=compartment_id)
    delete_vcn(quick_start_vcn_ocid=quick_start_vcn_ocid,
               vcn_client=vcn_client)


if __name__ == "__main__":
    main()
```

実行する時は、こんな形で実行できます。

```bash
python3 delete-oke-quickstart-vcn.py \
  --compartment-id <compartment-id> \
  --vcn-id <vcn-id>
```

`--help` オプションをつけて実行すれば、何となく使い方が分かると思います。

```bash
python3 delete-oke-quickstart-vcn.py --help
usage: delete-oke-quickstart-vcn.py [-h] [--compartment-id COMPARTMENT_ID] [--vcn-id VCN_ID] [--profile PROFILE]

OKE quick-start VCN automatically deleting script.

options:
  -h, --help                       show this help message and exit
  --compartment-id COMPARTMENT_ID  [required]
  --vcn-id VCN_ID                  [required]
  --profile PROFILE                [optional]
```

# 終わりに

動かない、等ありましたら本記事のコメントもしくは、[Issue](https://github.com/shukawam/oci-scripts/issues) を上げてもらえば対応します。
