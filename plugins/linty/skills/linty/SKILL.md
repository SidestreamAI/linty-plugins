---
name: linty
description: Find food, beverage and pet brands that sell direct with the Linty MCP server. See where to apply for a wholesale account, and read what each brand asks for. Use it when the user wants to stock a brand or open a wholesale account. Use it to make a list of brands to buy from.
---

# Linty

Linty is a public index of the wholesale programs of food, beverage and pet brands. The Linty MCP server reads the same records as linty.xyz. The free tools need no key. The account tools need a key.

## Make a list of brands

1. To see the categories, call `list_categories`. Send a `path` to see the categories under it. Each count is for published brands.
   A category has a page when it has 10 published brands. Without a page, `category` is null, and the result still lists the categories under the path.
2. Call `find_brands` with `category`, `country`, `availability` or `q` (text). Send `submittable: true` to keep only the brands with a web form at their address.
3. When `next_cursor` is not null, call `find_brands` again with `cursor` set to it, to get the next page.
4. For each brand, call `get_relationship` with the brand slug. The result says where to apply and lists the fields and documents that the brand asks for.

## Read the facts

- `availability` has four values. `direct`: the brand sells direct. `indirect_only`: the brand sells only through distributors. `unavailable`: the brand does not sell wholesale. `unknown`: Linty has no evidence.
- `unknown` is a complete answer. Do not guess. Tell the user that Linty has no evidence.
- Each fact has a source and a date. Give the user the date with the fact.
- A requirement is often "not verified": Linty saw the field on the form, but no person confirmed it.
- For the full facts of one brand, call `get_brand`. For its company, call `get_company`.

## Use an account

The account tools need a key. The user makes a key at linty.xyz/account and sets it in `LINTY_API_KEY`. Without a key, the server lists only the free tools. If a tool answers `invalid_key`, or the account tools are missing, the key in `LINTY_API_KEY` is missing, revoked or wrong. Tell the user to make a key at linty.xyz/account and set the variable. Never ask the user to paste the key in the chat.

1. Call `get_account` to see the credits and the profile values that a first application needs.
2. Call `get_applicant_profile` to read the profile. The EIN and the resale certificate number are masked: never send a masked value back.
3. Ask the user for each missing value. Then call `update_applicant_profile` with `fields`, `positioning` or `answers`. Never invent a value. Never send the EIN or the resale certificate number in `answers`. Send them in `fields`.
4. To add a document, calculate the SHA-256 and the size in bytes of the file. Call `get_upload_url` with them as `sha256` and `size_bytes`. Then send the same bytes with an HTTP PUT to `upload_url`. If `get_applicant_profile` shows `file_check` as `mismatch` or `missing`, upload the file again.
5. Call `check_application` for a brand. It lists the required fields and documents that the profile cannot answer.

## What Linty does not do yet

Linty does not submit applications yet. `submit_application` answers `submission_not_open`. Give the user the address of the form, portal or email of the brand, from `get_relationship`.

## Report a wrong fact

When the user finds a wrong fact, call `report_correction`. Send the brand slug, and what is wrong and what is right. If the fact is about a relationship, send the relationship slug too. A person at Linty reads each correction.
