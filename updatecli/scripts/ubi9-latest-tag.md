# Red Hat Container Catalog API Documentation

## Overview

The script fetches the latest tag from the Red Hat Container Catalog API for UBI9 images. It ensures that `jq` and `curl` are installed, fetches the tags, and processes them to find the unique tag.

## API Description

The Red Hat Container Catalog API, which provides the tag information, is described in the Swagger API documentation:
[https://catalog.redhat.com/api/containers/v1/ui/#/Repositories/graphql.images.get_images_by_repo](https://catalog.redhat.com/api/containers/v1/ui/#/Repositories/graphql.images.get_images_by_repo)

We would use:
- `registry.access.redhat.com` in the `registry` field
- `ubi9` in the `repository` field
- `100` in the `page_size` field
- `0` in the `page` field
- `last_update_date[desc]` in the `sort_by` field

### Parameters
- registry: `registry.access.redhat.com`  
  - This specifies the registry from which we are fetching the images. In this case, it is the Red Hat Container Catalog.
- repository: `ubi9`  
  - This specifies the repository within the registry. Here, we are interested in the UBI9 (Universal Base Image 9) repository.
- page_size: `100`  
  - This sets the number of results to return per page. We set it to 100 to ensure we get a sufficient number of tags in one request.
- page: `0`  
  - This specifies the page number to fetch. We start with page 0 to get the first set of results.
- sort_by: `last_update_date[desc]`  
  - This sorts the results by the `last_update_date` field in descending order. This ensures that the most recently updated tags are listed first.

### Curl Command
The curl command is used to make an HTTP GET request to the Red Hat Container Catalog API with the specified parameters. It fetches the JSON data containing the tags for the UBI9 images.
Click on "Execute" to get a curl command such as:

```sh
curl -X 'GET' \
  'https://catalog.redhat.com/api/containers/v1/repositories/registry/registry.access.redhat.com/repository/ubi9/images?page_size=100&page=0&sort_by=last_update_date%5Bdesc%5D' \
  -H 'accept: application/json'
``` 
- `-X 'GET'`: Specifies the HTTP method to use, which is GET in this case.
- `URL`: The URL constructed with the parameters to fetch the UBI9 image tags.
- `-H 'accept: application/json'`: Sets the request header to accept JSON responses.

The result would then contain something like:

```json
{
  "added_date": "2024-11-28T15:34:55.940000+00:00",
  "name": "latest",
  "_links": {
    "tag_history": {
      "href": "/v1/tag-history/registry/registry.access.redhat.com/repository/ubi9/tag/latest"
    }
  }
}
```

That's the beginning of what we're looking for. This will guide us to the tag considered the latest by RedHat. 
If we look closely at the preceding sibling in the JSON file, we'll find something like :

```json
{
  "added_date": "2024-11-28T15:34:55.940000+00:00",
  "name": "9.5-1732804088",
  "_links": {
    "tag_history": {
      "href": "/v1/tag-history/registry/registry.access.redhat.com/repository/ubi9/tag/9.5-1732804088"
    }
  }
},
{
  "added_date": "2024-11-28T15:34:55.940000+00:00",
  "name": "latest",
  "_links": {
    "tag_history": {
      "href": "/v1/tag-history/registry/registry.access.redhat.com/repository/ubi9/tag/latest"
    }
  }
}
```
What lies in the value associated to the `"name"` key in the first data structure is what we're looking for.
That's why in the script we are searching for all values associated to the `"name"` key in the data block where we found `"latest"`:

```bash
latest_tag=$(echo "$response" | jq -r '.data[].repositories[] | select(.tags[].name == "latest") | .tags[] | select(.name != "latest" and (.name | contains("-"))) | .name' | sort -u)
``` 

As you can see, we focus on the values that contain `-`, because we're only interested in the long-form tag name (`9.5-1732804088` and not `9.5`). We'll also keep only one instance of the tag, just in case the JSON would contain duplicates.

```bash
unique_tag=$(echo "$latest_tag" | sort | uniq | grep -v latest | grep "-")
```
