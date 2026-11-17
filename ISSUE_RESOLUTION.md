# Issue Resolution: ChainableUndefined Causes 'is escaped' Test to Pass (#2244)

## 1. Upstream Issue Metadata
- **Repository:** `pallets/jinja`
- **Issue Reference:** [#2244 - ChainableUndefined causes `is escaped` test to pass for `undefined` child elements](https://github.com/pallets/jinja/issues/2244)
- **Component:** `src/jinja2/runtime.py` (`ChainableUndefined`), `src/jinja2/tests.py` (`test_escaped`)
- **Severity / Impact:** High (Template Logic Corruption / Unexpected Safe-HTML Branching). In applications configuring `undefined=ChainableUndefined` with autoescaping enabled (e.g. frontend design systems such as NHS.UK Frontend Jinja port), evaluating `{{ obj.child is escaped }}` or `{{ undefined_var is escaped }}` incorrectly returned `True`. This caused templates to falsely bypass escaping or branch into unsafe HTML blocks intended only for pre-escaped markup.

---

## 2. Root Cause Analysis
1. `ChainableUndefined` defined a custom method:
   ```python
   def __html__(self) -> str:
       return str(self)
   ```
2. The Jinja built-in test `test_escaped` was implemented as:
   ```python
   def test_escaped(value: t.Any) -> bool:
       return hasattr(value, "__html__")
   ```
3. Because `ChainableUndefined` implemented `__html__`, any undefined chained attribute or variable (e.g. `parent.child`) possessed `__html__`.
4. As a result, `parent.child is escaped` evaluated to `True`. Regular `Undefined`, `StrictUndefined`, and `DebugUndefined` do not define `__html__`, making this an inconsistent anomaly in `ChainableUndefined`.
5. Moreover, undefined variables are not escaped markup and should never satisfy the `is escaped` predicate.

---

## 3. Reproduction Test
In `tests/test_tests.py`:
```python
    def test_escaped_chainable_undefined(self):
        env = Environment(undefined=ChainableUndefined, autoescape=True)
        tmpl = env.from_string(
            "{{ x.y is escaped }}|{{ x is escaped }}|{{ y is escaped }}"
        )
        assert tmpl.render(y=Markup("foo")) == "False|False|True"
```

### Pre-Fix Failure Output:
```text
FAILED tests/test_tests.py::TestTestsCase::test_escaped_chainable_undefined
E       AssertionError: assert 'True|True|True' == 'False|False|True'
E         - False|False|True
E         + True|True|True
```

---

## 4. Implementation Diff
```diff
diff --git a/src/jinja2/runtime.py b/src/jinja2/runtime.py
index 667c041..74683e2 100644
--- a/src/jinja2/runtime.py
+++ b/src/jinja2/runtime.py
@@ -985,9 +985,6 @@ class ChainableUndefined(Undefined):
 
     __slots__ = ()
 
-    def __html__(self) -> str:
-        return str(self)
-
     def __getattr__(self, name: str) -> "ChainableUndefined":
         # Raise AttributeError on requests for names that appear to be unimplemented
         # dunder methods to avoid confusing Python with truthy non-method objects that
diff --git a/src/jinja2/tests.py b/src/jinja2/tests.py
index 1a59e37..6c69642 100644
--- a/src/jinja2/tests.py
+++ b/src/jinja2/tests.py
@@ -202,7 +202,8 @@ def test_iterable(value: t.Any) -> bool:
 
 def test_escaped(value: t.Any) -> bool:
     """Check if the value is escaped."""
-    return hasattr(value, "__html__")
+    return hasattr(value, "__html__") and not isinstance(value, Undefined)
```

---

## 5. Verification Results
- **Reproduction Test:** `pytest tests/test_tests.py -k test_escaped_chainable_undefined` passed (1 passed).
- **Test Tests Suite:** `pytest tests/test_tests.py` passed (107 passed in 0.23s).
- **Full Test Suite:** `pytest tests/` passed (912 passed in 2.68s, 0 regressions).
- **Ruff Lint Check:** `ruff check src/jinja2/runtime.py src/jinja2/tests.py tests/test_tests.py` passed with 0 errors.
