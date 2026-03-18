# Cloudinary SDK Capabilities Research
**Technical Research Report**  
**Date:** February 23, 2026  
**User:** Edgar  
**Project:** AwsMigration  
**Language:** Python  
**Skill Level:** Intermediate

---

## Executive Summary

This research validates that the **Cloudinary Python SDK provides comprehensive capabilities** for your migration requirements:

✅ **Asset Filtering:** Full support—filter by name, date, and size using Search API or Admin API  
✅ **Asset Listing:** Multiple methods available with pagination and sorting  
✅ **Asset Updates:** Tags, context metadata, and structured metadata all supported  
✅ **URL Management:** Programmatic retrieval; moving to S3 requires download + S3 upload workflow  

**Recommendation:** Cloudinary SDK is fully capable for your filtering and tagging requirements. For S3 migration, combine Cloudinary SDK with boto3 for seamless asset transfer.

---

## 1. Filtering & Listing Assets

### 1.1 Filter by Name (Public ID)

**Search API** (Recommended for complex queries)
```python
from cloudinary import Search

# Search by exact public_id
result = (Search()
    .expression("public_id='my_image'")
    .max_results(100)
    .execute())

assets = result.get("resources", [])
for asset in assets:
    print(f"Found: {asset['public_id']}")
```

**Admin API** (Simple prefix/listing)
```python
import cloudinary.api as api

# List all assets with a specific prefix
result = api.resources(prefix="folder/subfolder", max_results=100)
resources = result.get("resources", [])
```

**Wildcard Search**
```python
# Search for public_ids starting with "product_"
result = (Search()
    .expression("public_id:product_*")
    .execute())
```

---

### 1.2 Filter by Date

**Uploaded Date Range**
```python
from cloudinary import Search

# Assets uploaded in the last 7 days
result = (Search()
    .expression("uploaded_at>7d")
    .max_results(100)
    .execute())

# Assets uploaded between two specific dates
result = (Search()
    .expression('uploaded_at:["2026-01-01" TO "2026-02-23"]')
    .max_results(100)
    .execute())

# Assets created after a specific date
result = (Search()
    .expression("created_at>'2026-02-01'")
    .sort_by("uploaded_at", "desc")
    .max_results(100)
    .execute())
```

**Date Field Options**
- `uploaded_at`: When asset was uploaded to Cloudinary
- `created_at`: When asset was first created in system
- `last_updated`: Last modification timestamp
- Format: ISO 8601 (`"2026-02-23T14:00:00Z"`) or relative (`7d`, `4w`, `1m`)

---

### 1.3 Filter by Size (Bytes)

**File Size Ranges**
```python
from cloudinary import Search

# Assets larger than 1 MB
result = (Search()
    .expression("bytes>1000000")  # or bytes>1mb
    .max_results(100)
    .execute())

# Assets between 500 KB and 5 MB
result = (Search()
    .expression("bytes:[500000 TO 5000000]")  # or bytes:[500kb TO 5mb]
    .execute())

# Small assets (less than 100 KB)
result = (Search()
    .expression("bytes<100000")
    .execute())
```

**Size Unit Suffixes**
- `b` = bytes (default)
- `kb` = kilobytes
- `mb` = megabytes
- `gb` = gigabytes

---

### 1.4 Combined Filtering Example

```python
from cloudinary import Search

# Assets matching: name contains "campaign", size > 500KB, uploaded in last 30 days
result = (Search()
    .expression(
        'public_id:campaign* AND bytes>500000 AND uploaded_at>30d'
    )
    .sort_by("uploaded_at", "desc")
    .sort_by("bytes", "asc")
    .max_results(500)
    .execute())

total_count = result.get("total_count", 0)
print(f"Found {total_count} matching assets")

for asset in result.get("resources", []):
    print(f"  {asset['public_id']}: {asset['bytes']} bytes, "
          f"uploaded {asset['uploaded_at']}")
```

---

### 1.5 Pagination

```python
from cloudinary import Search

# Get first 50 results
result = (Search()
    .expression("resource_type:image")
    .max_results(50)
    .execute())

resources = result.get("resources", [])
next_cursor = result.get("next_cursor")

# Fetch next batch if available
if next_cursor:
    next_result = (Search()
        .expression("resource_type:image")
        .max_results(50)
        .next_cursor(next_cursor)
        .execute())
    resources.extend(next_result.get("resources", []))
```

---

## 2. Updating Assets

### 2.1 Adding Tags

**Add Tag to Existing Asset**
```python
import cloudinary.api as api

# Add single tag
api.add_tag("processed", public_ids=["my_image_1", "my_image_2"])

# Add multiple tags to one asset
api.update("my_image", tags="processed,approved,client_review")

# Remove tag
api.remove_tag("old_tag", public_ids=["my_image_1"])
```

---

### 2.2 Adding Context (Simple Metadata)

**Context = Key-Value Pairs**
```python
import cloudinary.api as api

# Add contextual metadata (key-value pairs)
api.update(
    "my_image",
    context={
        "alt": "Beautiful sunset over mountains",
        "caption": "Location: Swiss Alps",
        "author": "Edgar",
        "status": "approved",
        "review_date": "2026-02-23"
    }
)

# Retrieve asset with context
asset = api.resource("my_image", context=True)
print(asset.get("context", {}))
```

---

### 2.3 Structured Metadata (Advanced)

**Metadata Field Types Supported**
- `string`: Text field
- `integer`: Numeric field
- `date`: Date field (format: yyyy-mm-dd)
- `enum`: Single-select from predefined list
- `set`: Multi-select from predefined list

**Workflow: Define Schema, Then Assign Values**

```python
import cloudinary.api as api

# Step 1: Create a metadata field definition
# (Usually done once, not per asset)
field_def = api.add_metadata_field({
    "type": "enum",
    "external_id": "approval_status",
    "label": "Approval Status",
    "mandatory": False,
    "datasource": {
        "values": [
            {"external_id": "pending", "value": "Pending"},
            {"external_id": "approved", "value": "Approved"},
            {"external_id": "rejected", "value": "Rejected"}
        ]
    }
})

# Step 2: Assign metadata value to asset
api.update(
    "my_image",
    metadata={"approval_status": "approved"}
)

# Step 3: Retrieve asset with metadata
asset = api.resource("my_image", metadata=True)
print(asset.get("metadata", {}))
```

**List All Metadata Fields**
```python
import cloudinary.api as api

fields = api.list_metadata_fields()
for field in fields.get("metadata_fields", []):
    print(f"Field: {field['external_id']} (type: {field['type']})")
```

---

### 2.4 Update Example: Combined Tags + Context + Metadata

```python
import cloudinary.api as api

public_id = "product_photo_2026"

# Single call updates tags, context, and metadata
api.update(
    public_id,
    tags="ecommerce,seasonal,2026",
    context={
        "product_name": "Winter Jacket Pro",
        "category": "Apparel",
        "price": "99.99",
        "in_stock": "true"
    },
    metadata={
        "approval_status": "approved",
        "quality_score": "95"
    }
)
```

---

## 3. Managing URLs & Asset Storage

### 3.1 Retrieving Asset URLs

```python
import cloudinary.api as api

# Get asset details including URLs
asset = api.resource("my_image")

print(f"Public URL: {asset['url']}")
print(f"Secure URL: {asset['secure_url']}")
print(f"Size: {asset['bytes']} bytes")
print(f"Format: {asset['format']}")
print(f"Resource Type: {asset['resource_type']}")
```

---

### 3.2 Transferring Assets from Cloudinary to S3

**⚠️ Key Finding:** Cloudinary does NOT have a single API call to "move" assets to external storage. Workflow:

1. Download from Cloudinary (via URL)
2. Upload to S3
3. Optionally delete from Cloudinary

**Complete Implementation**

```python
import cloudinary.api as api
import requests
import boto3
from typing import List, Dict

def migrate_assets_to_s3(
    public_ids: List[str],
    s3_bucket: str,
    s3_prefix: str = "assets/",
    delete_after_migrate: bool = False
) -> Dict[str, Dict]:
    """
    Migrate assets from Cloudinary to AWS S3.
    
    Args:
        public_ids: List of Cloudinary public_ids to migrate
        s3_bucket: Target S3 bucket name
        s3_prefix: S3 path prefix (default: "assets/")
        delete_after_migrate: Delete from Cloudinary after successful S3 upload
    
    Returns:
        Dict with migration status for each asset
    """
    
    s3_client = boto3.client("s3")
    results = {}
    
    for public_id in public_ids:
        try:
            # 1. Get asset details from Cloudinary
            asset = api.resource(public_id)
            url = asset['secure_url']
            format_ext = asset['format']
            
            # 2. Download asset from Cloudinary
            response = requests.get(url, stream=True)
            response.raise_for_status()
            
            # 3. Upload to S3
            s3_key = f"{s3_prefix}{public_id}.{format_ext}"
            
            s3_client.upload_fileobj(
                response.raw,
                s3_bucket,
                s3_key,
                ExtraArgs={
                    "ContentType": response.headers.get("Content-Type"),
                    "Metadata": {
                        "cloudinary_public_id": public_id,
                        "migrated_date": "2026-02-23"
                    }
                }
            )
            
            print(f"✓ Migrated {public_id} to s3://{s3_bucket}/{s3_key}")
            
            # 4. Optional: Delete from Cloudinary after successful S3 upload
            if delete_after_migrate:
                api.delete_resources([public_id])
                print(f"  → Deleted from Cloudinary")
            
            results[public_id] = {
                "status": "success",
                "s3_uri": f"s3://{s3_bucket}/{s3_key}",
                "s3_key": s3_key
            }
            
        except Exception as e:
            print(f"✗ Failed to migrate {public_id}: {str(e)}")
            results[public_id] = {
                "status": "failed",
                "error": str(e)
            }
    
    return results


# Usage Example
if __name__ == "__main__":
    # Set up credentials (ensure AWS_ACCESS_KEY_ID, AWS_SECRET_ACCESS_KEY env vars set)
    import os
    
    os.environ["CLOUDINARY_URL"] = "cloudinary://your_api_key:your_api_secret@your_cloud_name"
    
    # Migrate specific assets
    assets_to_migrate = ["product_1", "product_2", "product_3"]
    
    results = migrate_assets_to_s3(
        public_ids=assets_to_migrate,
        s3_bucket="my-migration-bucket",
        s3_prefix="cloudinary-assets/",
        delete_after_migrate=False  # Set True only after verifying S3 uploads
    )
    
    # Report results
    successful = [k for k, v in results.items() if v["status"] == "success"]
    failed = [k for k, v in results.items() if v["status"] == "failed"]
    
    print(f"\n--- Migration Summary ---")
    print(f"Successful: {len(successful)}/{len(assets_to_migrate)}")
    print(f"Failed: {len(failed)}/{len(assets_to_migrate)}")
    
    if failed:
        print(f"\nFailed assets: {', '.join(failed)}")
```

---

### 3.3 Batch Migration with Filtering

```python
import cloudinary.api as api
from cloudinary import Search

def migrate_filtered_assets_to_s3(
    filter_expression: str,
    s3_bucket: str,
    s3_prefix: str = "assets/"
):
    """
    Find assets matching criteria, then migrate to S3.
    
    Args:
        filter_expression: Lucene-like search expression
        s3_bucket: Target S3 bucket
        s3_prefix: S3 path prefix
    """
    
    # Find all matching assets
    search = Search().expression(filter_expression).max_results(500)
    result = search.execute()
    
    public_ids = [asset["public_id"] for asset in result.get("resources", [])]
    
    print(f"Found {len(public_ids)} assets matching filter: {filter_expression}")
    
    # Migrate them
    from migration_script import migrate_assets_to_s3
    results = migrate_assets_to_s3(public_ids, s3_bucket, s3_prefix)
    
    return results


# Example: Migrate all "processed" tagged assets larger than 500KB from last 30 days
if __name__ == "__main__":
    results = migrate_filtered_assets_to_s3(
        filter_expression='tags=processed AND bytes>500000 AND uploaded_at>30d',
        s3_bucket="migration-bucket",
        s3_prefix="processed-assets/"
    )
```

---

## 4. Implementation: Setup & Dependencies

### 4.1 Python Environment Setup

```bash
# Install required packages
pip install cloudinary boto3 requests python-dotenv

# Or add to requirements.txt
cloudinary==1.36.0
boto3==1.34.0
requests==2.31.0
python-dotenv==1.0.0
```

### 4.2 Configuration

```python
# .env file
CLOUDINARY_CLOUD_NAME=your_cloud_name
CLOUDINARY_API_KEY=your_api_key
CLOUDINARY_API_SECRET=your_api_secret

AWS_ACCESS_KEY_ID=your_aws_key
AWS_SECRET_ACCESS_KEY=your_aws_secret
AWS_REGION=us-east-1
```

```python
# config.py
import os
from dotenv import load_dotenv
import cloudinary

load_dotenv()

# Configure Cloudinary
cloudinary.config(
    cloud_name=os.getenv("CLOUDINARY_CLOUD_NAME"),
    api_key=os.getenv("CLOUDINARY_API_KEY"),
    api_secret=os.getenv("CLOUDINARY_API_SECRET")
)
```

---

## 5. Capability Matrix

| Capability | Supported | Method | Notes |
|---|---|---|---|
| **Filter by name/public_id** | ✅ | `Search().expression("public_id:...")` | Exact or wildcard patterns |
| **Filter by upload date** | ✅ | `Search().expression("uploaded_at>30d")` | ISO 8601 or relative times |
| **Filter by file size** | ✅ | `Search().expression("bytes>1mb")` | Supports byte/kb/mb/gb units |
| **Filter by tags** | ✅ | `Search().expression("tags=mytag")` | Exact or tokenized search |
| **Get asset list** | ✅ | `Search().execute()` or `api.resources()` | Up to 500 results per call w/ pagination |
| **Pagination** | ✅ | `next_cursor` parameter | Handle large result sets efficiently |
| **Add tags** | ✅ | `api.add_tag()` or `api.update(..., tags=...)` | Single or batch operations |
| **Add context (simple metadata)** | ✅ | `api.update(..., context={...})` | Key-value pairs |
| **Add structured metadata** | ✅ | `api.update(..., metadata={...})` | Requires schema definition first |
| **Retrieve URLs** | ✅ | `api.resource().get('secure_url')` | Both HTTP and HTTPS available |
| **Direct S3 reassignment** | ❌ | N/A | Use download + S3 upload workflow |
| **Batch operations** | ✅ | Loop + `api.update()` or search | Efficient for hundreds of assets |

---

## 6. Rate Limits & Performance Considerations

| Operation | Rate Limit | Notes |
|---|---|---|
| Admin API (general) | 500 requests/hour | Shared across all Admin calls |
| Search API | No hard limit (Tier 1) | Subject to account plan |
| Upload API | Based on plan | Separate from Admin API limit |
| Metadata operations | Included in Admin limit | Create fields once, reuse often |

**Best Practices:**
- Batch operations when possible (e.g., `api.add_tag()` accepts multiple public_ids)
- Cache metadata field definitions
- Use pagination for large result sets
- Implement exponential backoff for rate limit retries

---

## 7. Recommendations for AWS Migration

### 7.1 Migration Strategy

1. **Asset Discovery Phase**
   - Use Search API to identify assets by criteria
   - Apply filters: date range, size, tags, etc.
   - Export public_ids to CSV for tracking

2. **Data Transfer Phase**
   - Implement parallel batch downloads (50-100 concurrent)
   - Use S3 multipart upload for large files
   - Store S3 paths in Cloudinary context metadata for cross-reference

3. **Verification Phase**
   - Compare checksums (ETag) between Cloudinary and S3
   - Verify file counts and sizes match
   - Test S3 URL accessibility

4. **Cleanup Phase**
   - Optionally delete from Cloudinary (after verification)
   - Update application code to use S3 URLs
   - Monitor for any broken reference issues

### 7.2 Sample Migration Script Template

```python
import logging
from datetime import datetime

# Configure logging
logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s'
)
logger = logging.getLogger(__name__)

def migrate_campaign():
    """
    Full migration workflow for a specific campaign.
    """
    
    logger.info("Starting migration workflow...")
    
    # Step 1: Filter assets
    logger.info("Step 1: Discovering assets...")
    search_expr = (
        'tags=campaign_2026 AND '
        'uploaded_at>["2026-01-01" TO "2026-02-23"] AND '
        'bytes>100000'
    )
    
    # Step 2: Migrate
    logger.info("Step 2: Migrating to S3...")
    results = migrate_filtered_assets_to_s3(
        filter_expression=search_expr,
        s3_bucket="my-bucket",
        s3_prefix="campaign_2026/"
    )
    
    # Step 3: Report
    successful = sum(1 for v in results.values() if v["status"] == "success")
    logger.info(f"✓ Migration complete: {successful}/{len(results)} successful")
```

---

## 8. Conclusion & Next Steps

### ✅ Confirmed Capabilities

- **Cloudinary SDK fully supports asset filtering** by name, date, and size
- **Tag and metadata management** is straightforward and efficient
- **S3 migration is feasible** using download + upload pattern with boto3

### 🔄 Next Steps

1. **Prototype Phase:**
   - Test filtering with your real Cloudinary account
   - Verify metadata field definitions work for your data
   - Run small batch migration (10-50 assets) to S3

2. **Production Readiness:**
   - Implement error handling and retry logic
   - Set up CloudWatch logging for monitoring
   - Validate S3 access controls and encryption

3. **Documentation:**
   - Create runbooks for your team
   - Document metadata schema
   - Establish migration checkpoints

---

## 9. Architecture Clarification: Cloudinary as Management-Only Proxy

**Session Update:** 2026-02-23 (Architecture Validation)

### The Correct Three-Layer Architecture

Your project uses a **management-only proxy** model, NOT a storage replacement or Fetch API model:

```
User Requests
    ↓
CloudFront (public URL, HTTPS edge delivery)
    ↓
S3 (physical asset storage)
    ↓
Cloudinary (metadata, versioning, transformations metadata only - NOT serving files)
```

**Key Points:**
1. **S3 stores the physical asset** - source of truth for bytes
2. **CloudFront delivers** - user-facing public URL  
3. **Cloudinary manages** - metadata, tags, versioning, historical records

### How to Move Existing Assets to S3 (Premium Account with External Storage Backend)

**Assumption:** Your Cloudinary account has:
- ✅ Premium plan (or higher)
- ✅ External Storage Backend feature enabled and configured to your S3 bucket
- ✅ Account-level configuration pointing to S3

**With External Storage Backend enabled, you have a clean path:**

#### For New Uploads (Post-Configuration)

```python
import cloudinary.uploader

# After External Storage Backend is configured at account level,
# new uploads automatically go to S3 (not Cloudinary storage)

response = cloudinary.uploader.upload(
    "path/to/new_file.jpg",
    public_id="new_asset_001"
)

# Result:
# ✓ File stored in configured S3 bucket (not Cloudinary)
# ✓ Metadata managed by Cloudinary (searchable, taggable)
# ✓ You pay S3 storage, NOT Cloudinary storage
# ✓ secure_url still points to res.cloudinary.com (Cloudinary proxies from S3)
```

---

#### For Existing Assets (Migration to S3)

To move existing Cloudinary-stored assets to S3, **re-upload them** using `overwrite=True`:

```python
import cloudinary.api
import cloudinary.uploader
import requests
import boto3
from typing import List, Dict

def migrate_existing_asset_to_s3_backend(public_id: str) -> Dict:
    """
    Migrate an existing Cloudinary-stored asset to S3 via External Storage Backend.
    
    Workflow:
    1. Download from Cloudinary
    2. Re-upload with overwrite=True (goes to configured S3 backend)
    3. Cloudinary stores new version in S3 instead of replacing in-place
    """
    
    try:
        # Step 1: Get existing asset metadata
        asset = cloudinary.api.resource(public_id)
        cloudinary_url = asset["secure_url"]
        original_format = asset.get("format", "jpg")
        
        # Step 2: Download from Cloudinary
        response = requests.get(cloudinary_url, timeout=30)
        response.raise_for_status()
        
        # Step 3: Re-upload with overwrite=True
        # Since External Storage Backend is configured, this goes to S3
        result = cloudinary.uploader.upload(
            response.content,
            public_id=public_id,
            overwrite=True,           # Replace old version
            invalidate=True,          # Clear CDN cache
            resource_type="auto",     # Auto-detect type
            context={
                "migration_status": "moved_to_s3_backend",
                "migration_timestamp": "2026-02-23T10:00:00Z",
                "original_cloudinary_storage": "true"
            },
            tags=["s3_backend_migrated"]
        )
        
        print(f"✓ {public_id}: re-uploaded to S3 backend")
        
        return {
            "status": "success",
            "public_id": public_id,
            "new_version": result.get("version"),
            "storage_backend": "s3",  # Now stored in S3
            "url": result.get("secure_url")
        }
        
    except Exception as e:
        print(f"✗ Migration failed for {public_id}: {str(e)}")
        return {
            "status": "failed",
            "public_id": public_id,
            "error": str(e)
        }


def migrate_batch_to_s3_backend(public_ids: List[str], concurrent_workers: int = 5) -> Dict:
    """
    Migrate multiple existing assets to S3 backend in parallel.
    """
    from concurrent.futures import ThreadPoolExecutor, as_completed
    
    results = {}
    
    with ThreadPoolExecutor(max_workers=concurrent_workers) as executor:
        futures = {
            executor.submit(migrate_existing_asset_to_s3_backend, pid): pid 
            for pid in public_ids
        }
        
        for future in as_completed(futures):
            pid = futures[future]
            result = future.result()
            results[pid] = result
    
    return results


# Usage Example
if __name__ == "__main__":
    import os
    from dotenv import load_dotenv
    import cloudinary
    
    load_dotenv()
    cloudinary.config(
        cloud_name=os.getenv("CLOUDINARY_CLOUD_NAME"),
        api_key=os.getenv("CLOUDINARY_API_KEY"),
        api_secret=os.getenv("CLOUDINARY_API_SECRET")
    )
    
    # Migrate specific assets
    assets_to_migrate = ["product_1", "product_2", "product_3"]
    
    results = migrate_batch_to_s3_backend(
        public_ids=assets_to_migrate,
        concurrent_workers=5
    )
    
    # Report
    successful = [k for k, v in results.items() if v["status"] == "success"]
    failed = [k for k, v in results.items() if v["status"] == "failed"]
    
    print(f"\n--- Migration Summary ---")
    print(f"Successful: {len(successful)}/{len(assets_to_migrate)}")
    print(f"Failed: {len(failed)}/{len(assets_to_migrate)}")
    
    if failed:
        print(f"\nFailed assets: {', '.join(failed)}")
```

---

#### Migrate Filtered Assets to S3 Backend

```python
from cloudinary import Search

def migrate_filtered_to_s3_backend(
    filter_expression: str,
    concurrent_workers: int = 5
) -> Dict:
    """
    Find assets by criteria, then migrate all to S3 backend.
    """
    
    # Find matching assets
    search = Search().expression(filter_expression).max_results(500)
    result = search.execute()
    
    public_ids = [asset["public_id"] for asset in result.get("resources", [])]
    total_count = result.get("total_count", 0)
    
    print(f"Found {len(public_ids)} assets matching filter")
    print(f"Total count available: {total_count} (pagination needed for all)")
    
    # Migrate them
    return migrate_batch_to_s3_backend(public_ids, concurrent_workers)


# Example: Migrate all images larger than 500KB from last 30 days
if __name__ == "__main__":
    results = migrate_filtered_to_s3_backend(
        filter_expression='resource_type:image AND bytes>500000 AND uploaded_at>30d',
        concurrent_workers=5
    )
```

---

### What Happens After Migration to S3 Backend

**Before (Asset in Cloudinary storage):**
```
Your App
  ↓
https://res.cloudinary.com/my-cloud/.../product_1.jpg
  ↓
Cloudinary servers (serves from own datacenter)
  ↓
Image downloaded
```

**After (Asset in S3 backend):**
```
Your App
  ↓
https://res.cloudinary.com/my-cloud/.../product_1.jpg  ← Same public URL!
  ↓
Cloudinary (fetches from S3, applies transforms if requested)
  ↓
S3 bucket (source of truth)
  ↓
Optionally: CloudFront in front for CDN acceleration
```

**Key Points:**
- ✅ Public URL **remains the same** (`res.cloudinary.com/...`)
- ✅ Cloudinary still manages metadata, tags, transformations
- ✅ Storage moved to S3 (you own the data)
- ✅ Cloudinary proxies from S3 transparently
- ✅ Can add CloudFront on top for additional CDN layer

---

### Account-Level Configuration Checklist

Before running the migration code, verify:

```python
import cloudinary.api

# Check account settings
account = cloudinary.api.account_info()

# Look for:
# - Storage backend type: "s3" or similar
# - External storage configured: True
# - S3 bucket name
# - Permissions configured

print(account)

# Verify External Storage Backend is enabled:
# Settings > Upload > External Storage Backend > S3
# (Admin dashboard)
```

# (Admin dashboard)
```

If External Storage Backend is NOT showing as enabled:
1. Contact Cloudinary support (premium feature, may need activation)
2. Provide S3 bucket name and IAM credentials
3. They'll configure account-level integration
4. Test with small upload after configuration

---

### S3 Folder Structure & Naming Conventions

#### Cloudinary Limitations on Public IDs

When migrating existing assets, be aware of Cloudinary's public_id constraints:

| Character | Allowed | Notes |
|---|---|---|
| `/` (forward slash) | ✅ | Creates nested structure (e.g., `folder/subfolder/image`) |
| `.` (dot) | ❌ | Breaks Cloudinary URL parsing; avoid in public_id |
| `_` (underscore) | ✅ | Safe; use as dot replacement |
| `-` (hyphen) | ✅ | Safe; use as dot replacement |
| Spaces | ❌ | URL encoding issues |
| Other special chars | ❌ | Limit to alphanumeric + `_`, `-`, `/` |

**Migration Rule:** If original Cloudinary public_id has dots, replace with underscores:
```
Original: product.line.item.jpg        → Cloudinary: product_line_item.jpg
S3 path:  products/lines/items/...jpg  → S3 supports dots freely
```

#### Recommended S3 Folder Structure

With External Storage Backend configured, structure your S3 bucket by strategy:

**Strategy 1: Chronological Organization**
```
s3://my-bucket/
├── cloudinary-assets/2026/
│   ├── january/
│   │   ├── product_001_1.jpg
│   │   ├── product_002_1.jpg
│   │   └── campaign_launch_hero.jpg
│   ├── february/
│   │   └── product_003_1.jpg
│   └── march/
└── archived/
    └── old_campaigns/
```

**Strategy 2: By Asset Type**
```
s3://my-bucket/
├── cloudinary-assets/
│   ├── images/
│   │   ├── ecommerce/
│   │   ├── blog/
│   │   └── social/
│   └── videos/
│       ├── tutorials/
│       └── ads/
```

**Strategy 3: By Cloudinary Tag (recommended for searchability)**
```
s3://my-bucket/
├── cloudinary-assets/
│   ├── ecommerce/     # Matches Cloudinary tag: ecommerce
│   ├── blog/          # Matches Cloudinary tag: blog
│   ├── social/        # Matches Cloudinary tag: social
│   └── archived/      # Matches Cloudinary tag: archived
```

#### Store Mapping in Cloudinary Context

Regardless of S3 structure, always document the mapping:

```python
import cloudinary.api

cloudinary.api.update(
    "product_001",
    context={
        "storage_backend": "aws_s3",
        "s3_bucket": "my-bucket",
        "s3_key": "cloudinary-assets/ecommerce/product_001.jpg",
        "s3_uri": "s3://my-bucket/cloudinary-assets/ecommerce/product_001.jpg",
        "s3_folder_strategy": "by-tag",  # Document your choice
        "migration_date": "2026-02-23"
    },
    tags=["s3_migrated", "ecommerce"]
)
```

---

### New File Uploads After Migration (Frontend Upload)

#### Default Cloudinary Folder Structure

When uploading via Cloudinary frontend (web upload widget) **before** External Storage Backend:
- **Default location:** Root or `sample` folder
- **Generated public_id:** `bcyzd3zppwwjs1a2b3c4d5e` (random if not specified)
- **Storage location:** Cloudinary datacenter
- **URL:** `https://res.cloudinary.com/my-cloud/image/upload/bcyzd3zppwwjs1a2b3c4d5e.jpg`

#### After External Storage Backend Configuration

Once External Storage Backend is configured and enabled on your account:

**Frontend upload (no folder specified):**
```
Upload via Cloudinary web widget
    ↓
Cloudinary receives file
    ↓
Upload to configured S3 backend
    ↓
Default S3 location (account-level config)
```

**S3 destination depends on account-level configuration:**
- **Common default:** `uploads/` prefix
- **Or custom:** `my-org/incoming/` or similar (configured by you with Cloudinary support)

**Example paths:**
```
s3://my-bucket/uploads/bcyzd3zppwwjs1a2b3c4d5e.jpg
s3://my-bucket/incoming/photo_2026_02_23.jpg
```

#### To Control Frontend Upload Location

Use Cloudinary Upload Widget with folder parameter:

```html
<!-- HTML upload widget (after External Storage Backend enabled) -->
<script src="https://upload-widget.cloudinary.com/global/all.js"></script>
<script>
  cloudinary.createUploadWidget(
    {
      cloudName: "my-cloud",
      uploadPreset: "my_upload_preset",
      folder: "new-uploads/2026/february",  // ← Specifies S3 subfolder
      publicId: "auto"  // Auto-generate public_id
    },
    (error, result) => {
      if (!error && result && result.event === "success") {
        console.log("File uploaded to: " + result.info.public_id);
      }
    }
  ).open();
</script>
```

**With `folder` parameter:**
- Frontend upload goes to: `s3://my-bucket/new-uploads/2026/february/{filename}`
- Cloudinary public_id: `new-uploads/2026/february/filename`
- Context automatically added (if configured)

#### Python Upload (Programmatic) After Migration

```python
import cloudinary.uploader

# New upload after External Storage Backend enabled
response = cloudinary.uploader.upload(
    "path/to/new_file.jpg",
    folder="new-uploads/2026",  # S3 subfolder
    public_id="product_004",     # Results in: new-uploads/2026/product_004
    context={
        "uploaded_via": "frontend_widget",
        "upload_date": "2026-02-23",
        "storage_backend": "aws_s3"
    }
)

# Result in S3:
# s3://my-bucket/new-uploads/2026/product_004.jpg
```

---

### Migration vs. New Upload Storage Paths

| Scenario | Storage Location | S3 Path | Public ID | Notes |
|---|---|---|---|---|
| **Existing migrated asset** | S3 (migrated path) | `cloudinary-assets/ecommerce/product_001.jpg` | `product_001` | Depends on migration strategy |
| **New upload via frontend** | S3 (default) | `uploads/random_id.jpg` | `random_id` | Uses account-level default |
| **New upload with folder** | S3 (specified folder) | `new-uploads/2026/product_004.jpg` | `new-uploads/2026/product_004` | Folder parameter controls path |
| **Programmatic upload** | S3 (specified folder) | `custom/path/file.jpg` | `custom/path/file` | Use folder param in SDK |

---

