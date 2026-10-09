---
name: linty
description: Find food, beverage and pet brands that sell direct with the Linty MCP server. See where to apply for a wholesale account, and read what each brand asks for. Use it when the user wants to stock a brand or open a wholesale account. Use it to make a list of brands to buy from. With an account, Linty can submit the application to the form of the brand.
---

# Linty

Linty is a public index of the wholesale programs of food, beverage and pet brands. The Linty MCP server reads the same records as linty.xyz. The free tools need no key. The account tools need a key.

## Make a list of brands

1. To see the categories, call `list_categories`. Send a `path` to see the categories under it. Each count is for published brands.
   A category has a page when it has 10 published brands. Without a page, `category` is null, and the result still lists the categories under the path.
2. Call `find_brands` with `category`, `country`, `availability` or `q` (text). Send `submittable: true` to keep only the brands with an active web form at their address that Linty can use.
3. When `next_cursor` is not null, call `find_brands` again with `cursor` set to it, to get the next page.
4. For each brand, call `get_relationship` with the brand slug. The result says where to apply and lists the fields, documents and terms that the brand asks for.

## Read the facts

- `availability` has four values. `direct`: the brand sells direct. `indirect_only`: the brand sells only through distributors. `unavailable`: the brand does not sell wholesale. `unknown`: Linty has no evidence.
- `unknown` is a complete answer. Do not guess. Tell the user that Linty has no evidence.
- Each fact has a source and a date. Give the user the date with the fact.
- Each requirement has `seen`: where Linty saw it, on the brand's page or in an application, and the date. A fact from the brand's page has its quote in `source`.
- `last_checked_at` is the date that LintyBot last read the brand's page. A read is not a verification.
- Read `terms` before the user applies. It says whether Linty read the agreement that the form asks the user to accept. Give the user `terms.note`.
- A requirement is often "not verified": Linty saw the field on the form, but no person confirmed it.
- For the full facts of one brand, call `get_brand`. For its company, call `get_company`.

## Use an account

The account tools need a key. The user makes a key at linty.xyz/account and sets it in `LINTY_API_KEY`. Without a key, the server lists only the free tools. If a tool answers `invalid_key`, or the account tools are missing, the key in `LINTY_API_KEY` is missing, revoked or wrong. Tell the user to make a key at linty.xyz/account and set the variable. Never ask the user to paste the key in the chat.

1. Call `get_account` to see the credits and the profile values that a first application needs.
2. Call `get_applicant_profile` to read the profile. The EIN and the resale certificate number are masked: never send a masked value back.
3. Ask the user for each missing value. Then call `update_applicant_profile` with `fields`, `positioning` or `answers`. Never invent a value. Never send the EIN or the resale certificate number in `answers`. Send them in `fields`. Ask the user the unit of the orders: `truckload`, `pallet`, `case` or `unit`. Send it in `order_volume_unit`.
4. To add a document, calculate the SHA-256 and the size in bytes of the file. Call `get_upload_url` with them as `sha256` and `size_bytes`. Then send the same bytes with an HTTP PUT to `upload_url`. If `get_applicant_profile` shows `file_check` as `mismatch` or `missing`, upload the file again.
5. Call `check_application` for a brand. It lists the required fields and documents that the profile cannot answer.

## Submit an application

Linty submits each application to the form of the brand, in the name of the user. The user cannot recall an application. Ask the user before each submission.

Submission is open only to some accounts now. Another account gets `submission_not_open`. Then give the user the address of the form, portal or email of the brand, from `get_relationship`.

1. Call `check_application` for the brand. If `submittable` is false, Linty cannot apply to this brand. If `existing_application` is not null, the user has an application to this brand already: read it with `get_application`.
2. If `missing_fields` or `missing_documents` is not empty, get the values or the files from the user first.
3. If `attestations` is not empty, show the user each agreement before you submit. Show its `label`, its `agreement_url` and its `quote`. Show each `related` restriction with its `quote`.
4. Linty accepts each agreement with `linty_accepts` true for the user when it submits. For a required agreement with `linty_accepts` false, the application asks the user later, in `questions`.
5. If `terms.status` is not `read`, Linty did not read the agreement of the form. Give the user `terms.note`.
6. Call `submit_application` with the brand slug. Send `answers` for form questions that the profile has no field for.
7. Keep the `application_id`. If `existing` is true, the user has an application to this brand already, and Linty made no second one.
8. Call `get_application` to see the status. Call `list_applications` to see every application of the account.

An application holds 1 credit while it is in progress. Linty uses the credit only when the form of the brand shows a success message. Every other end gives the credit back.

If `submit_application` answers an error, read `code`:

- `profile_incomplete`: `missing` lists the profile values to add with `update_applicant_profile`.
- `no_credit`: the account has no credit left. Tell the user to email hello@linty.xyz.
- `daily_cap`: the account sent its daily maximum. Try again after `resets_at`.
- `brand_busy`: Linty sent this brand its maximum for 7 days. Try again after `retry_after` seconds.
- `not_submittable`: `reason` says why Linty cannot apply to this brand. For `walled`, Linty saw a bot check on the form that its browser cannot pass. For `site_policy`, a `robots.txt` file asks automated programs not to use the page or a part of it. For `not_read`, Linty has no read of the page of the form that `robots.txt` allows, so Linty does not apply there now. Give the user the address of the form from `get_relationship`, so that the user can apply on the site of the brand.

## What to do with each status

- `queued` or `running`: Linty is at work. Call `get_application` again after 5 minutes.
- `needs_answers`: read `questions`. Ask the user each question, and never invent an answer.
  - For a question of the kind `document`, upload the file with `get_upload_url` first. If the profile has that file already, do not upload it again.
  - For a question whose `key` is a profile field, send the value with `update_applicant_profile` in `fields`. That value replaces the profile value for each later form too.
  - A `business_type` question has `options`, and none of them agrees with the business type of the profile. Show the options to the user. Send an option only if it is the business of the user. If the user picks `Other`, tell the user that Linty then selects `Other` on each later form that has it. If no option is the business of the user, cancel the application.
  - For a question of the kind `attestation`, the form asks the user to accept an agreement. Show the user its `label`, and ask the user to accept it. Send `yes` only if the user accepts it. Linty keeps that answer for this application only. If the user does not accept it, cancel the application with `cancel_application`. Linty sends nothing to the brand, and the credit comes back.
  - Then call `answer_questions` with the other answers, or with no answers. Linty queues the application again.
- `submitted`: the form showed a success message. The brand replies to the email address of the profile. `accepted_attestations` lists the agreements that Linty accepted for the user.
- `needs_followup`: the brand asks the user to do an action. Tell the user `followup.action` and `followup.deadline`.
- `unconfirmed`: Linty sent the form, but the page showed no success message. The brand can have the application. Linty does not send it again.
  - Tell the user to look in the inbox of the profile email for a reply from the brand.
  - After 7 days with no reply from the brand, the user can release the application. Ask the user first. Then call `release_application`, and apply again with `submit_application`.
  - If the brand has the first application, the second one reaches it too. `unconfirmed_cause` says why Linty could not confirm the first one.
- `not_submitted`: the credit is back. Read `reason`.
  - `blocked`: a bot check on the form of the brand stopped Linty. Linty sent nothing. Tell the user, and give the address of the form from `get_relationship`. The user can apply on the site of the brand.
  - `unsupported`: the form asks for a value or a file that Linty cannot send yet. Linty sent nothing. Tell the user, and give the address of the form.
  - `released`: the user released an unconfirmed application. You can call `submit_application` again for this brand.
  - `no_form_found`: Linty found no form that it can fill. Linty sent nothing. Call `check_application` before you call `submit_application` again: the form can be one that Linty does not apply to now.
  - For another reason, Linty sent nothing, and you can call `submit_application` again for this brand.
- `cancelled`: the user cancelled the application.

To stop an application, call `cancel_application`. It works only while the status is `queued` or `needs_answers`.

## Report a wrong fact

When the user finds a wrong fact, call `report_correction`. Send the brand slug, and what is wrong and what is right. If the fact is about a relationship, send the relationship slug too. A person at Linty reads each correction.
