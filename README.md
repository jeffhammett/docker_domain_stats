# docker_domain_stats
Contains dockerfile to build domain_stats.py as an image

### Description

This docker image runs **domain_stats.py**. This is a python service that is designed to perform mass domain analysis. It can do things such as find the creation_date of a domain and identify if a domain is a member of the Alexa/Cisco Umbrella top 1 million sites.

It was developed to be used in conjunction with a SIEM and is in production environments. Specifically, it has been used in conjunction with the Elastic Stack, such as queried by Logstash, with large success. 

### To run the docker image
```sh
sudo docker run -d --name container_name_goes_here -p 20000:20000 lightforge/domain_stats
```

### Updating Top-1m file
The docker image does not currently automatically update the top-1m.csv. The below example shows how to download a new top 1 million site list and have a domain_stats container use it. This could be scheduled as a cron job on your host to keep a current Alexa/Cisco Umbrella top-1m.csv in use.

```sh
wget -q http://s3.amazonaws.com/alexa-static/top-1m.csv.zip; unzip top-1m.csv.zip;
sudo docker cp top-1m.csv container_name_goes_here:/opt/domain_stats/top-1m.csv
sudo docker restart so-domainstats
```

### Caution - SIEM Use
Keep in mind that doing domain information lookups can add around a **half second** to log ingestion per log. This is an extreme penalty but is offset by domain_stats.py's caching capabilities. For example, perfomring a creation_date lookup on a single domain such as hasecuritysolutions.com would take approximately a half second for the initial log but additional lookups would be near instantanious.

One recommended way to minimize cache use is to only perform WHOIS lookups if a domain is not a top 1 million site. The Logstash code to do this is below:

```js
filter {
  if [type] == "dns" {
    if [highest_registered_domain] {
      rest {
        request => {
          url => "http://localhost:20000/alexa/%{highest_registered_domain}"
        }
        sprintf => true
        json => false
        target => "site"
      }
      if [site] != "0" and [site] {
        mutate {
          add_tag => [ "top-1m" ]
          remove_field => [ "site" ]
        }
      }
    }
    if "alexa" not in [tags] and [query] !~ "\.internal\.domain$" and [highest_registered_domain] and [highest_registered_domain] != "" {
      rest {
        request => {
          url => "http://localhost:20000/domain/creation_date/%{highest_registered_domain}"
        }
        sprintf => true
        json => false
        target => "creation_date"
      }
      if [creation_date] and [creation_date] !~ "No whois record"{
        date {
          match => [ "creation_date", "YYYY-MM-dd HH:mm:ss'; '",
                                      "YYYY-MM-dd'T'HH:mm:ssZ'; '",
                                      "YYYY-MM-dd'T'HH:mm:ss'.00Z; '" ]
          target => "creation_date"
        }
      }
    }
  }
}
```
