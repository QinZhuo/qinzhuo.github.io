# 可搜索下拉框
基于AdvancedDropdown制作的可搜索下拉框


![](https://raw.githubusercontent.com/QinZhuo/qinzhuo.github.io/refs/heads/master/unity/qdropdown.png)
```csharp
public class QAdvancedDropdown : AdvancedDropdown {
	public AdvancedDropdownItem Root { get; private set; }
	public QDictionary<string,AdvancedDropdownItem> Items { get; private set; }
	public QDictionary<AdvancedDropdownItem, Action> Actions { get; private set; } = new();
	public IMouseEvent MouseEvent { get; set; }
	public QAdvancedDropdown(string title, Action<string> onItemSelected = null) : base(new AdvancedDropdownState()) {
		Root = new AdvancedDropdownItem(title);
		Items = new(key => {
			if (key.Contains('/')) {
				var index = key.LastIndexOf('/');
				var start = key.Substring(0, index);
				var end = key.Substring(index+1);
				var item = new AdvancedDropdownItem(end);
				Items[start].AddChild(item);
				if (onItemSelected != null) {
					Actions[item] = () => onItemSelected(key);
				}
				return item;
			}
			else {
				var item = new AdvancedDropdownItem(key);
				Root.AddChild(item);
				if (onItemSelected != null) {
					Actions[item] = () => onItemSelected(key);
				}
				return item;
			}
		});

	}
	public void AddSeparator() {
		Root.AddSeparator();
	}
	public void Add(string key,Action action=null) {
		var item = Items[key];
		if (action != null) {
			Actions[item] = action;
		}
	}
	public void Add(IList<string> keys) {
		foreach (var key in keys) {
			Add(key);
		}
	}
	protected override AdvancedDropdownItem BuildRoot() {
		return Root;
	}
	protected override void ItemSelected(AdvancedDropdownItem item) {
		base.ItemSelected(item);
		Actions[item]?.Invoke();
	}
}
```
```csharp
	public static void AddSearchMenu(this VisualElement root, Action<QAdvancedDropdown> menuBuilder, float? width = null) {
		root.RegisterCallback<MouseUpEvent>(e => {
			if (e.button == 1) {
				var dropdown = new QAdvancedDropdown();
				dropdown.MouseEvent = e;
				menuBuilder(dropdown);
				if (width.HasValue) {
					dropdown.Show(new Rect { center = e.mousePosition, width = width.Value, height = 0 });
				}
				else {
					dropdown.Show(root.worldBound);
				}
				e.StopPropagation();
			}
		});
	}
```
```csharp
	[CustomPropertyDrawer(typeof(QPopupAttribute))]
	public class QPopupDrawer : PropertyDrawer
	{
		public override VisualElement CreatePropertyGUI(SerializedProperty property)
		{
			var root = new VisualElement();
			root.style.flexDirection = FlexDirection.Row;
			var popupData = QPopupData.Get(property, (attribute as QPopupAttribute).getListFuncs);
			var value = property.propertyType == SerializedPropertyType.String ? property.stringValue : property.objectReferenceValue?.GetType()?.Name;
			if (value == null) {
				value = "\t";
			}
			if (!popupData.List.Contains(value))
			{
				popupData.List.Add(value);
			}
			if (property.propertyType == SerializedPropertyType.ObjectReference && value == null)
			{
				value = popupData.List.Get(0);
			}

			//var visual = new PopupField<string>(QReflection.QName(property), popupData.List, value);
			//root.Add(visual);
			var propertyField = root.Add(property);
			propertyField.RegisterCallback<ClickEvent>(e => {
				var dropdown = new QAdvancedDropdown(QReflection.QName(property), key => {
					if (property.propertyType == SerializedPropertyType.String) {
						property.stringValue = key;
					}
					else {
						var gameObject = (property.serializedObject.targetObject as MonoBehaviour)?.gameObject;
						if (gameObject != null) {
							value = key;
							if (value.IsNull()) {
								property.objectReferenceValue = null;
							}
							else {
								try {
									property.objectReferenceValue = gameObject.GetComponent(QReflection.ParseType(value), true);
								}
								catch (Exception) {
									throw;
								}
							}
						}
					}
					property.serializedObject.ApplyModifiedProperties();
				});
				dropdown.Add(popupData.List);
				dropdown.Show(propertyField.worldBound);
			});
		
			return root;
		}

		public override void OnGUI(Rect position, SerializedProperty property, GUIContent label) {
			EditorGUI.PropertyField(position, property, label);
		}
	}
```