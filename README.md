Add-DnsServerResourceRecordCName `
    -Name "nlb-proxy" `
    -HostNameAlias "internal-nlb-xxx.elb.region.amazonaws.com" `
    -ZoneName "corp.local"
