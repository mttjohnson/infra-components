# An acme-dns client for registering

Meant to be compatible with acme-dns-client so that the registration process can be scripted without being prompted for input from a TTY.

This should write to the json file `/etc/acmedns/clientstorage.json` that acme-dns-client uses for issuing a cert along with certbot.

## Manually call acme-dns API using curl

Example response from acme-dns /register endpoint:
```json
{"username":"2eeb12f6-5c07-46dd-9ae7-c6ebe4d5aec4","password":"FCYuJVPRNH0o8uHYmCMiO3PGc5tppt8_PpYTq2rC","fulldomain":"24ce2e33-ff4c-403f-89e7-7a5d76456e6d.my-acme-dns-server.example.com","subdomain":"24ce2e33-ff4c-403f-89e7-7a5d76456e6d","allowfrom":[]}
```

call /register
```bash
domain_for_registration="my-host.example.com"

acme_dns_api_url="https://my-acme-dns-server.example.com"
acme_dns_register=$(curl -s -X POST "${acme_dns_api_url}/register")
acme_dns_register='{"username":"2eeb12f6-5c07-46dd-9ae7-c6ebe4d5aec4","password":"FCYuJVPRNH0o8uHYmCMiO3PGc5tppt8_PpYTq2rC","fulldomain":"24ce2e33-ff4c-403f-89e7-7a5d76456e6d.my-acme-dns-server.example.com","subdomain":"24ce2e33-ff4c-403f-89e7-7a5d76456e6d","allowfrom":[]}'


acme_dns_register='bad response that isnt json'

# Conditional check to see if response has a valid subdomain
[[ $(echo -E "${acme_dns_register}" | jq 'has("subdomain")') = "true" ]] && echo 'was true'

acme_dns_client_new_entry=$(echo -E "${acme_dns_register}" | jq \
    --arg acme_dns_api_url "${acme_dns_api_url}" \
    --arg domain_for_registration "${domain_for_registration}" \
'{
    ($domain_for_registration): {
        fulldomain: .fulldomain,
        subdomain: .subdomain,
        username: .username,
        password: .password,
        server_url: $acme_dns_api_url
    }
}'
)

acme_dns_client_storage="/etc/acmedns/clientstorage.json"

cat "${acme_dns_client_storage}" <(echo "${acme_dns_client_new_entry}") | jq -s add


cp -a "${acme_dns_client_storage}" "${acme_dns_client_storage}_bak_$(date +"%Y%m%d%H%M%S")"
cat "${acme_dns_client_storage}" <(echo "${acme_dns_client_new_entry}") | jq -s add > "${acme_dns_client_storage}_new"
mv "${acme_dns_client_storage}_new" "${acme_dns_client_storage}"


```


it needs to be like this:
```json
{
  "my-host.example.com": {
    "fulldomain": "4e00ba5d-99fd-425d-bc62-93065c445f4f.my-acme-dns-server.example.com",
    "subdomain": "4e00ba5d-99fd-425d-bc62-93065c445f4f",
    "username": "81f44e22-3b8b-40ca-acf9-71c7656a85b5",
    "password": "xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx",
    "server_url": "https://my-acme-dns-server.example.com"
  },
  "xxxxx_domain_name_xxxxx": {
    "fulldomain": "xxxxx_assist_req_full_domain_xxxxx",
    "subdomain": "xxxxx_assist_req_sub_domain_xxxxx",
    "username": "xxxxx_assist_req_api_user_xxxxx",
    "password": "xxxxx_assist_req_api_pass_xxxxx",
    "server_url": "xxxxx_assist_req_api_url_xxxxx"
  }
}
```


## Remove an entry from /etc/acmedns/clientstorage.json 

```bash
domain_to_remove="my-host.example.com"
acme_dns_client_storage="/etc/acmedns/clientstorage.json"
acme_dns_reg_registered_domains_dir="/etc/acme_dns_reg/domains"

cat "${acme_dns_client_storage}" | jq .
ls -la "${acme_dns_reg_registered_domains_dir}"

cp -a "${acme_dns_client_storage}" "${acme_dns_client_storage}_bak_$(date +"%Y%m%d%H%M%S")"
cat "${acme_dns_client_storage}" | jq "del(.\"${domain_to_remove}\")" > "${acme_dns_client_storage}_new"
mv "${acme_dns_client_storage}_new" "${acme_dns_client_storage}"
rm "${acme_dns_reg_registered_domains_dir}/${domain_to_remove}"
```
