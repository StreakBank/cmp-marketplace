# UI Recipes: Forms & Input

Copy-paste-ready form patterns with validation. All commonMain-compatible, design-token-compliant.

---

## Basic Form Layout

```kotlin
@Composable
fun <Feature>Form(
    onSubmit: () -> Unit,
    modifier: Modifier = Modifier,
) {
    Column(
        modifier = modifier
            .fillMaxWidth()
            .padding(Spacing.lg),
        verticalArrangement = Arrangement.spacedBy(Spacing.md),
    ) {
        // Form fields go here

        Spacer(Modifier.height(Spacing.md))

        Button(
            onClick = onSubmit,
            modifier = Modifier.fillMaxWidth(),
        ) {
            Text(stringResource(Res.string.<feature>_form_submit))
        }
    }
}
```

## OutlinedTextField with Validation

```kotlin
@Composable
fun ValidatedTextField(
    value: String,
    onValueChange: (String) -> Unit,
    label: String,
    error: String?,
    modifier: Modifier = Modifier,
) {
    OutlinedTextField(
        value = value,
        onValueChange = onValueChange,
        label = { Text(label) },
        isError = error != null,
        supportingText = error?.let { { Text(it) } },
        singleLine = true,
        modifier = modifier.fillMaxWidth(),
    )
}
```

## Password Field with Visibility Toggle

```kotlin
@Composable
fun PasswordField(
    value: String,
    onValueChange: (String) -> Unit,
    label: String,
    error: String?,
    modifier: Modifier = Modifier,
) {
    var visible by remember { mutableStateOf(false) }

    OutlinedTextField(
        value = value,
        onValueChange = onValueChange,
        label = { Text(label) },
        isError = error != null,
        supportingText = error?.let { { Text(it) } },
        singleLine = true,
        visualTransformation = if (visible) VisualTransformation.None else PasswordVisualTransformation(),
        trailingIcon = {
            IconButton(onClick = { visible = !visible }) {
                Icon(
                    imageVector = if (visible) Icons.Default.VisibilityOff else Icons.Default.Visibility,
                    contentDescription = stringResource(
                        if (visible) Res.string.<feature>_form_hide_password
                        else Res.string.<feature>_form_show_password,
                    ),
                )
            }
        },
        keyboardOptions = KeyboardOptions(keyboardType = KeyboardType.Password),
        modifier = modifier.fillMaxWidth(),
    )
}
```

## Form State in ViewModel

**Note:** Form state uses `MutableStateFlow.update{}` — not `stateIn()`. This is a legitimate exception because form state is locally-owned, user-modified state (not derived from repository data).

```kotlin
data class <Feature>FormState(
    val name: String = "",
    val email: String = "",
    val nameError: String? = null,
    val emailError: String? = null,
    val isSubmitting: Boolean = false,
)

class <Feature>FormViewModel(
    private val submit<Feature>: Submit<Feature>UseCase,
) : ViewModel() {

    private val _formState = MutableStateFlow(<Feature>FormState())
    val formState: StateFlow<<Feature>FormState> = _formState.asStateFlow()

    fun onNameChanged(name: String) {
        _formState.update { it.copy(name = name, nameError = null) }
    }

    fun onEmailChanged(email: String) {
        _formState.update { it.copy(email = email, emailError = null) }
    }

    fun submit() {
        val state = _formState.value
        val nameError = if (state.name.isBlank()) getString(Res.string.<feature>_form_name_required) else null
        val emailError = if (!state.email.contains("@")) getString(Res.string.<feature>_form_email_invalid) else null

        if (nameError != null || emailError != null) {
            _formState.update { it.copy(nameError = nameError, emailError = emailError) }
            return
        }

        viewModelScope.launch {
            _formState.update { it.copy(isSubmitting = true) }
            submit<Feature>(state.name, state.email)
                .onSuccess { /* navigate or show success */ }
                .onFailure { _formState.update { s -> s.copy(isSubmitting = false) } }
        }
    }
}
```

## Form Submission with Loading

```kotlin
@Composable
fun <Feature>FormScreen(viewModel: <Feature>FormViewModel = koinViewModel()) {
    val formState by viewModel.formState.collectAsStateWithLifecycle()

    Column(
        modifier = Modifier
            .fillMaxWidth()
            .padding(Spacing.lg),
        verticalArrangement = Arrangement.spacedBy(Spacing.md),
    ) {
        ValidatedTextField(
            value = formState.name,
            onValueChange = viewModel::onNameChanged,
            label = stringResource(Res.string.<feature>_form_name_label),
            error = formState.nameError,
        )

        ValidatedTextField(
            value = formState.email,
            onValueChange = viewModel::onEmailChanged,
            label = stringResource(Res.string.<feature>_form_email_label),
            error = formState.emailError,
        )

        Spacer(Modifier.height(Spacing.md))

        Button(
            onClick = viewModel::submit,
            enabled = !formState.isSubmitting,
            modifier = Modifier.fillMaxWidth(),
        ) {
            if (formState.isSubmitting) {
                CircularProgressIndicator(
                    modifier = Modifier.size(IconSize.sm),
                    strokeWidth = Spacing.xxs,
                    color = MaterialTheme.colorScheme.onPrimary,
                )
            } else {
                Text(stringResource(Res.string.<feature>_form_submit))
            }
        }
    }
}
```
