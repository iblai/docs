# External Credentials

<iframe width="560" height="315" src="https://www.youtube.com/embed/ongBQqb_KIM" title="YouTube video player" frameborder="0" allowfullscreen></iframe>

## Purpose

Configure and manage external credential providers (e.g., badging vendors) and map your platform-issued credentials to those providers so credentials are issued externally while remaining managed within your platform.

---

## Where to Find It

1. Click your **tenant name**.  
2. Go to **Advanced**.  
3. Scroll to the bottom to find:
   - **Provider Configuration**
   - **Credential Mapping**

---

## Provider Configuration

Use this to **add, edit, enable/disable, or delete** an external provider.

### Add or Edit a Provider

1. Open **Provider Configuration**.  
2. Enter:
   - **Provider name**
   - **Configuration** (provider-specific settings)
   - **Enabled** (toggle on/off)
3. Click **Save**.

### Notes

- Editing looks the same as adding, except fields are **pre-filled**.  
- You can **delete providers** you no longer need.  
- **Enabled providers** appear in the mapping step.

---

## Map Credentials to an External Provider

Once a provider is configured, map platform credentials to the provider’s templates.

### Create the Mapping

1. Open **Credential Mapping**.  
2. Select:
   - **Credential**  
     - From the list created in your platform (typically from the course overview page)
   - **Provider**
   - **External Template ID**  
     - The credential/template ID from the provider
3. *(Optional)* Add:
   - **Group ID**
   - **Additional metadata**
4. Click **Save**.

### What’s Editable

- **Credential name** and **provider** are fixed after creation.  
- Optional fields (e.g., **group ID**, **metadata**, **external template ID**) can be added or updated.

---

## How Issuance Works

- The credential is **created and managed** in your platform.  
- The mapping links it to the **external provider’s template**.  
- When issued, the credential is **issued by the external provider** using that mapping.

---

## Result

You can self-manage **external credential providers** and seamlessly issue credentials through them—**without duplicating workflows or leaving the platform**.
