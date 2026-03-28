- هنستخدم **Custom Tag Helper with View Component Composition** 
- **بنستخدم Tag Helper و View Component سوا عشان نعمل reusable component، بحيث الـ Tag Helper ياخد المحتوى والخصائص، والـ View Component يعرض الـ HTML النهائي.**

### نعمل ايه؟؟
#### 1. Create the ViewModel

First, create a `ModalViewModel` to hold all data needed for the modal.
**`ViewModels/ModalViewModel.cs`**
``` csharp
namespace YourProject.ViewModels;

public class ModalViewModel
{
    public string? Id { get; set; } = "deleteModal";
    public string? Title { get; set; } = "Delete";
    public string? Controller { get; set; }
    public string? Action { get; set; } = "Delete";
    public string? ItemNameFieldId { get; set; } = "itemName";
    public string? ItemIdFieldId { get; set; } = "itemId";
    public IHtmlContent? InnerContent { get; set; }
}
```

Here's a detailed breakdown of each property in the `ModalViewModel`:

| Property | Type | Purpose | Why We Created It |
|----------|------|---------|-------------------|
| `Id` | `string?` | Unique identifier for the modal element (e.g., `deleteModal`). | To allow multiple modals on the same page without conflicts. The `Id` is used as the HTML `id` attribute for the modal container, which Bootstrap's JavaScript uses to show/hide the modal. |
| `Title` | `string?` | Title displayed in the modal header (e.g., "Delete Category"). | To customize the modal's title based on context (e.g., "Delete Product", "Delete User"). The title can include an icon or additional text. |
| `Controller` | `string?` | Name of the controller that handles the delete request. | To make the modal reusable across different controllers. The delete form’s `asp-controller` attribute uses this value to target the correct controller. |
| `Action` | `string?` | Name of the action method that processes the delete request. | To allow flexibility in naming the delete action (could be `Delete`, `DeleteConfirmed`, etc.). The delete form’s `asp-action` uses this. |
| `ItemNameFieldId` | `string?` | The `id` of the HTML element that will display the name of the item being deleted. | To decouple the modal’s JavaScript from fixed element IDs. This property lets the parent page specify which element should receive the item’s name when the modal opens. |
| `ItemIdFieldId` | `string?` | The `id` of the hidden input element that will store the ID of the item being deleted. | Similar to `ItemNameFieldId`, this allows the parent page to define where the item’s ID should be placed in the modal form. This ensures the delete form submits the correct ID. |
| `InnerContent` | `IHtmlContent?` | Arbitrary HTML content that will be rendered inside the modal body, **above** the hidden input field. | To allow full customization of the modal’s body content. With this property, the modal can display warnings, additional messages, or even complex layouts without modifying the modal’s core structure. |

#### 2. Create the View Component:
##### a) View Component Class
**`ViewComponents/DeleteModalViewComponent.cs`**
```
namespace YourProject.ViewComponents;

public class DeleteModalViewComponent : ViewComponent
{
    public IViewComponentResult Invoke(ModalViewModel model)
    {
        return View(model);
    }
}
```

##### b) ### View Component View
Create the folder `Views/Shared/Components/DeleteModal` and inside add `Default.cshtml`:
**`Views/Shared/Components/DeleteModal/Default.cshtml`**
``` html
@model ModalViewModel

<div class="modal fade" id="@Model.Id" tabindex="-1" 
     data-item-name-field="@Model.ItemNameFieldId" 
     data-item-id-field="@Model.ItemIdFieldId">
    <div class="modal-dialog modal-dialog-centered" role="document">
        <div class="modal-content">
            <div class="modal-header">
                <h5 class="modal-title text-danger"><i class="bi bi-exclamation-triangle-fill me-2"></i>@Model.Title</h5>
                <button type="button" class="btn-close shadow-none" data-bs-dismiss="modal"></button>
            </div>
            <form asp-controller="@Model.Controller" asp-action="@Model.Action" method="post">
                <div class="modal-body">
                    @Model.InnerContent
                    <input type="hidden" name="id" id="@Model.ItemIdFieldId" />
                </div>
                <div class="modal-footer">
                    <button type="button" class="btn btn-secondary shadow-none px-4" data-bs-dismiss="modal">Cancel</button>
                    <button type="submit" class="btn btn-danger shadow-none px-4">Delete</button>
                </div>
            </form>
        </div>
    </div>
</div>
```


#### 3. Create the TagHelper
**`TagHelpers/ModalTagHelper.cs`**
``` csharp
namespace Swipe.MVC.TagHelpers;

[HtmlTargetElement("modal")]
public class ModalTagHelper : TagHelper
{
	// Properties
    public string Id { get; set; } = "deleteModal";
    public string Title { get; set; } = "Delete";
    public string Controller { get; set; }
    public string Action { get; set; } = "Delete";
    public string ItemNameFieldId { get; set; } = "itemName";
    public string ItemIdFieldId { get; set; } = "itemId";

    [ViewContext]
    [HtmlAttributeNotBound]
    public ViewContext ViewContext { get; set; }

    private readonly IViewComponentHelper _viewComponentHelper;

    public ModalTagHelper(IViewComponentHelper viewComponentHelper)
    {
        _viewComponentHelper = viewComponentHelper;
    }

    public override async Task ProcessAsync(TagHelperContext context, TagHelperOutput output)
    {
        // Get the inner content between <modal> and </modal>
        TagHelperContent childContent = await output.GetChildContentAsync();
        string innerHtml = childContent.GetContent();

        // Build the model
        ModalViewModel model = new()
        {
            Id = Id,
            Title = Title,
            Controller = Controller,
            Action = Action,
            ItemNameFieldId = ItemNameFieldId,
            ItemIdFieldId = ItemIdFieldId,
            InnerContent = new HtmlString(innerHtml)
        };

        // Give the view component helper the current ViewContext
        if (_viewComponentHelper is IViewContextAware contextAware)
            contextAware.Contextualize(ViewContext);

        // Invoke the view component and get its HTML
        IHtmlContent result = await _viewComponentHelper.InvokeAsync("DeleteModal", new { model });

        // Replace the <modal> tag with the generated HTML
        output.TagName = null;
        output.Content.SetHtmlContent(result);
    }
}
```

**Properties:**

```csharp
public string Id { get; set; } = "deleteModal";
public string Controller { get; set; }
// ... إلخ
```

دي الـ attributes اللي بتكتبها في الـ HTML:

```html
<modal controller="Products" action="Delete" title="حذف المنتج">
    هل متأكد؟
</modal>
```
---
**`[ViewContext]` + `[HtmlAttributeNotBound]`:**

```csharp
[ViewContext]
[HtmlAttributeNotBound]
public ViewContext ViewContext { get; set; }
```

- `[ViewContext]` → ASP.NET Core بيحقن الـ ViewContext تلقائياً
- `[HtmlAttributeNotBound]` → عشان المستخدم ميقدرش يمررها كـ HTML attribute
---
**`ProcessAsync`:**

```csharp
TagHelperContent childContent = await output.GetChildContentAsync(); // النص الداخلي جوه الـ tag
string innerHtml = childContent.GetContent();              // ex."هل متأكد؟"
```

بعدين بيبني `ModalViewModel`.

---
**الجزء المهم:**

```csharp
if (_viewComponentHelper is IViewContextAware contextAware)
    contextAware.Contextualize(ViewContext);
```

#### الـ `ViewContext` إيه؟

لما المستخدم يفتح صفحة في موقعك، ASP.NET بيجمع معلومات عن الطلب ده ويحطها في object اسمه `ViewContext`. زي:

- إنت في أنهي Controller؟
- إنت في أنهي Action؟
- إيه الـ URL الحالي؟
- إيه الـ ViewData؟

يعني الـ `ViewContext` = **بطاقة هوية الصفحة الحالية.**

---
#### المشكلة

الـ `IViewComponentHelper` بييجي من الـ DI — وبييجي **فاضي، مش عارف هو شغال في أنهي صفحة.**

تخيله زي GPS مش عارف موقعك الحالي — مش هيقدر يوديك حتة.

---
#### الحل

```csharp
if (_viewComponentHelper is IViewContextAware viewContextAware)
{
    viewContextAware.Contextualize(ViewContext);
}
```

**السطر الأول** — بيسأل: هل الـ helper ده بيفهم لو قولتله "أنت شغال هنا"؟ (مش كل object بيفهم، فبنتحقق الأول)

**السطر التاني** — بيقوله: "أنت شغال في الصفحة دي، الـ ViewContext بتاعتها ده."

زي ما تقول للـ GPS: **موقعي الحالي هو كذا** — دلوقتي يقدر يشتغل.

---
## بعد السطرين دول

```csharp
IHtmlContent result = await _viewComponentHelper.InvokeAsync("DeleteModal", model);
```

دلوقتي الـ helper عارف هو فين، وقدر يشتغل ويرجع الـ HTML.

---

الفكرة في جملة واحدة:

> زي ما تفتح برنامج على كمبيوتر — لازم تقوله أنت شغال فين الأول، عشان يعرف يجيب الملفات الصح.

```csharp
output.TagName = null; // بيشيل الـ <modal> tag نفسها من الـ output
output.Content.SetHtmlContent(result); // وبيحط مكانها الـ HTML الناتج
```

---
#### 4. Register the Tag Helper

Open `_ViewImports.cshtml` (located in the `Views` folder) and add:

```csharp
@addTagHelper *, YourProject //(assembly name not the namespace) 
```

---
#### 5. JavaScript code:
`js/site.js`
``` javascript
// Handle any modal trigger generically
$(document).on('click', '[data-bs-toggle="modal"]', function () {
    const $button = $(this);
    const $targetModal = $($button.data('bs-target'));

    if ($targetModal.length) {
        // Get the values from the button
        const id = $button.data('bs-id');
        const name = $button.data('bs-name');

        // Get the target field IDs from the modal's metadata
        const nameFieldId = $targetModal.data('item-name-field');
        const idFieldId = $targetModal.data('item-id-field');

        // Update the modal content
        if (nameFieldId) $targetModal.find('#' + nameFieldId).text(name);
        if (idFieldId) $targetModal.find('#' + idFieldId).val(id);
    }
});
```

---
### Usage:

1. create the `btn` 
``` html
<button type="button" 
        class="btn btn-sm btn-outline-danger" 
        data-bs-toggle="modal" 
        data-bs-target="#deleteModal" 
        data-bs-id="@category.Id" 
        data-bs-name="@category.EnglishName">
    <i class="bi bi-trash"></i> Delete
</button>
```

2. use the call Taghelper
``` html
<modal id="deleteModal"
       title="Delete Category"
       controller="Categories"
       action="Delete"
       item-name-field-id="deleteCategoryName"
       item-id-field-id="deleteCategoryId">
    <h2 class="text-warning"><i class="bi bi-exclamation-triangle-fill"></i> Warning!</h2>
    <p>Are you sure you want to delete the category '<strong id="deleteCategoryName"></strong>'?</p>
    <p class="text-muted small">This action cannot be undone.</p>
</modal>
```

