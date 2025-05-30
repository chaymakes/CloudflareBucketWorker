# Using R2 Bucket from a worker

Just some notes, and examples on using R2 Bucket from a worker; and how this works when developing  locally.

Based on bug fix: https://github.com/cloudflare/workers-sdk/issues/8868#issuecomment-2840098674

 1) Local vs Remote bucket.
 2) Configure wrangler, to bind buckets.
 3) Put resource asset into a bucket, and add meta data type.
 4) Get resource from a bucket, and checking the meta type.
 5) Process the resource base on its type; (eg) JSON or text.

### 1. Run local dev using:
```
npx wrangler dev --remote
```

Use **--remote**; enables the locally worker to access the data stored remotely on Cloudflare servers.
If the **-remote** is not use then wrangler will create local buckets on your computer, base on the **r2_buckets: []**
configuration in your "wrangler.jsonc" file. And the worker will use this local data store.

Note:
 - **bucket_name:**  is named bucket to use when worker is deployed live; (worker is also remote on the Cloudflare server)
 - **preview_bucket_name:** is used by the worker when running from local dev, but the remote data will come from an alternative but for testing.

To use the same bucket as the live, simple use the live bucket name for both **bucket_name**, and **preview_bucket_name**
However, note your locally app will than have access to over write your live data.
### 2. Example wrangler, with binding to R2 buckets, live and preview

wrangler.jsonc
```
{
	"$schema": "node_modules/wrangler/config-schema.json",
	"name": "buckets-test",
	"main": "src/index.js",
	"compatibility_date": "2025-04-29",
	"observability": {
		"enabled": true
	},

	"r2_buckets": [
		{
			"binding": "BUCKET",
			"bucket_name": "dataset",
			"preview_bucket_name": "datasetpreview"
		}
	]
}
```

### 3. Put examples

Use put() to add resource to the bukcet, this its meta data.
```
const ResourceName = "2025.json"

export default {
	async fetch(request, env, ctx) {
		// Put data into the bucket
		var date = new Date()
		var obj = {
			name: "chaypalton",
			site: "chaymakes.com",
			date: date.toLocaleDateString() + " " + date.toLocaleTimeString()
		}

		const objbucket = await env.BUCKET.put(ResourceName, JSON.stringify(obj),
			{
				httpMetadata: { contentType: "application/json" }
			})
	}
}

```
Get env from cloudflare Global is offset useful. For pages Functions.
```
import { env } from "cloudflare:workers"
export default {
	async fetch(request) {
	}
}
```

Worker:
```
const ResourceName = "2025.json"
export default {
	async fetch(request) {
		// Read the (file) resource, not inital we get the resource object.
		var R2Object = await env.BUCKET.get(name)
		var result = ""

		if (R2Object != undefined) {

			console.log("File " + name + ": contentType = " + R2Object.httpMetadata.contentType)

			if (R2Object.httpMetadata.contentType == "text/plain") {

				result = await R2Object.text()

			} else if (R2Object.httpMetadata.contentType == "application/json") {

				// Test only read JSON, than convert back to text
				var object = await R2Object.json()
				result = JSON.stringify(object)

			} else if (R2Object.range.length > 0) {

				// Test only assume text
				console.log("No contentType: Assuming text");

				try {
					result = await R2Object.text()
				} catch (e) {
					console.log(e)
				}
			}
		}

		if (R2Object == undefined || R2Object.httpMetadata.contentType == undefined) {
			console.log("---------------------------")
			console.log("Error reading bucket resource : " + name)
			console.log("---------------------------")
			console.log(R2Object)
		} else {
			console.log("---------------------------")
			console.log("OKAY: Reading bucket resource : " + name)
			console.log("---------------------------")
			console.log(result)
		}

		return new Response(result);

	},
};
```
