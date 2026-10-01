# Playwright E2E Test – Buyer Enquiry to Admin Leads

```typescript
import { expect, test } from "@playwright/test";

const CATALOGUE_URL = "/catalogue/premium-corporate-essentials";

const PRODUCT = {
  name: "Terra insulated flask",
  moq: 50,
  price: "₹649",
};

test("buyer enquiry shows up correctly in the admin Leads inbox", async ({
  browser,
}) => {
  // Test data
  const buyer = {
    name: `E2E Buyer ${Date.now().toString(36)}`,
    phone: "9876501234",
  };

  const quantity = PRODUCT.moq * 2;

  // Create separate browser pages for Buyer and Admin
  const buyerContext = await browser.newContext();
  const buyerPage = await buyerContext.newPage();

  const adminContext = await browser.newContext();
  const adminPage = await adminContext.newPage();

  // --------------------------------------------------
  // 1. Buyer opens the catalogue and selects a product
  // --------------------------------------------------

  await buyerPage.goto(CATALOGUE_URL);

  await expect(
    buyerPage.getByRole("heading", { level: 1 })
  ).toContainText("Premium corporate essentials");

  await buyerPage
    .getByRole("link", { name: PRODUCT.name })
    .first()
    .click();

  await expect(
    buyerPage.getByRole("heading", {
      level: 1,
      name: PRODUCT.name,
    })
  ).toBeVisible();

  // --------------------------------------------------
  // 2. Buyer fills the enquiry form and submits it
  // --------------------------------------------------

  await buyerPage.getByRole("button", { name: "Contact Us" }).click();

  const enquiryDialog = buyerPage.getByRole("dialog");

  await enquiryDialog.getByLabel("Your name").fill(buyer.name);

  await enquiryDialog
    .getByLabel("WhatsApp number")
    .fill(buyer.phone);

  await enquiryDialog
    .getByRole("button", { name: "Increase quantity" })
    .click();

  await expect(
    enquiryDialog.getByLabel("Quantity in units")
  ).toHaveValue(String(quantity));

  await enquiryDialog
    .getByRole("button", { name: "Send enquiry" })
    .click();

  // --------------------------------------------------
  // 3. Verify enquiry confirmation and get reference
  // --------------------------------------------------

  await expect(
    enquiryDialog.getByText("Enquiry sent")
  ).toBeVisible();

  const referenceElement = enquiryDialog.getByText(
    /^ENQ-\d{7}$/
  );

  await expect(referenceElement).toBeVisible();

  const enquiryReference = (
    await referenceElement.innerText()
  ).trim();

  // --------------------------------------------------
  // 4. Admin logs into the application
  // --------------------------------------------------

  await adminPage.goto("/login");

  await adminPage
    .getByLabel("Email")
    .fill("admin@catalogue.test");

  await adminPage
    .getByLabel("Password")
    .fill("Admin#2026");

  await adminPage
    .getByRole("button", { name: "Sign in" })
    .click();

  await expect(adminPage).toHaveURL(/\/admin/);

  // --------------------------------------------------
  // 5. Search for the enquiry in the Leads inbox
  // --------------------------------------------------

  await adminPage.goto("/admin/leads");

  await adminPage
    .getByPlaceholder("Search buyer, reference or contact")
    .fill(enquiryReference);

  const leadLink = adminPage.getByRole("link", {
    name: `View ${enquiryReference}`,
  });

  await expect(leadLink).toBeVisible();

  await leadLink.click();

  // --------------------------------------------------
  // 6. Verify lead details
  // --------------------------------------------------

  await expect(
    adminPage.getByRole("heading", {
      level: 1,
      name: enquiryReference,
    })
  ).toBeVisible();

  await expect(
    adminPage.getByText(buyer.name).first()
  ).toBeVisible();

  await expect(
    adminPage.getByText(PRODUCT.name).first()
  ).toBeVisible();

  await expect(
    adminPage.getByText(`${quantity} units`)
  ).toBeVisible();

  await expect(
    adminPage.getByText(PRODUCT.price)
  ).toBeVisible();

  await expect(
    adminPage.getByText(`+91${buyer.phone}`)
  ).toBeVisible();
});
```

## Test Flow

1. Buyer opens the premium corporate catalogue.
2. Buyer selects the Terra insulated flask.
3. Buyer opens **Contact Us** and enters name and WhatsApp number.
4. Quantity is increased from the MOQ to 100 units.
5. Buyer submits the enquiry.
6. Test captures the generated `ENQ-XXXXXXX` reference.
7. Admin logs into the application.
8. Admin searches the Leads inbox using the enquiry reference.
9. Test opens the lead and validates:
   - Enquiry reference
   - Buyer name
   - Product name
   - Quantity
   - Product price
   - WhatsApp number

## Framework Concepts Used

- Playwright Test
- `browser.newContext()` for isolated buyer/admin sessions
- Role-based locators
- Label-based locators
- Placeholder-based locator
- Regular expression locator
- `expect()` assertions
- Dynamic test data using `Date.now()`
- End-to-end business-flow validation
