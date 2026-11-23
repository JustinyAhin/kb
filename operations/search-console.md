# Google Search Console

Use this runbook to add a site to Search Console.

## Add the site

1. Add the site as a **Domain** property, for example `example.com`.
2. Verify it with the TXT record that Google gives. Put the record at the apex.
3. Open **Sitemaps**. Submit the full URL, for example `https://example.com/sitemap.xml`.
   A Domain property refuses `sitemap.xml` alone.

## Old domain

On a domain that had earlier owners, open **Sitemaps** and remove their sitemaps.

## Troubleshooting

| Symptom                            | Cause and fix                                               |
| ---------------------------------- | ----------------------------------------------------------- |
| The sitemap shows "Couldn't fetch" | This is frequent on a new property in the first days. Wait. |
| The sitemap form refuses the path  | The property is a Domain property. Submit the full URL.     |
