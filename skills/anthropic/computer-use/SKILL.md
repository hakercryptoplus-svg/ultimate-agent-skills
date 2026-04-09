---
  name: computer-use
  description: "Skills for controlling computer interfaces, browsers, and desktop UIs. Load when automating UI tasks, web testing, or visual automation."
  ---

  # Computer Use — استخدام الكمبيوتر كالإنسان

  ## Core Rules

  ### Screenshot First, Always
  NEVER interact with UI without taking a screenshot first. Look before you touch.

  ### Verify After Every Action
  ```
  Pattern for every interaction:
  1. take_screenshot() → analyze current state
  2. identify_target_element()
  3. perform_action()
  4. take_screenshot() → verify success
  5. handle_unexpected_state() if needed
  ```

  ## Element Identification (Best to Worst)
  ```
  1. Visible text ("Save", "Submit", "Create Account")
  2. Placeholder text in inputs
  3. ARIA labels
  4. Position relative to known elements
  5. Visual appearance — last resort
  ```

  ## Form Filling Pattern
  ```python
  async def fill_form_safely(fields: dict):
      for label, value in fields.items():
          click_on_label(label)
          keyboard.hotkey('ctrl', 'a')  # Select all
          keyboard.press('delete')       # Clear
          keyboard.type(str(value))      # Type new value
          
          # Verify entry
          screenshot = take_screenshot()
          assert str(value) in extract_text(screenshot)
  ```

  ## URL Safety Check Before Credentials
  ```python
  def verify_before_login(expected_domain: str):
      url = get_current_url()
      
      if not url.startswith('https://'):
          raise SecurityError("Not HTTPS — refusing to enter credentials")
      
      if expected_domain not in url:
          raise SecurityError(f"Wrong domain: {url}")
  ```

  ## Handling Popups
  ```python
  def dismiss_common_popups():
      screenshot = take_screenshot()
      text = extract_text(screenshot).lower()
      
      if any(word in text for word in ['cookie', 'gdpr', 'consent']):
          try_click_any(['Accept all', 'Accept', 'I agree', 'OK'])
      elif 'notification' in text:
          try_click_any(['Not now', 'No thanks', 'Block'])
      elif 'update' in text:
          try_click_any(['Skip', 'Later', 'Remind me later'])
  ```

  ## Scrolling to Find Elements
  ```python
  def scroll_to_find(target_text: str, max_scrolls=10):
      for _ in range(max_scrolls):
          if target_text in extract_text(take_screenshot()):
              return find_element_by_text(target_text)
          scroll_down(pixels=300)
      
      raise ElementNotFoundError(f"'{target_text}' not found after scrolling")
  ```

  ## NEVER Do Without Explicit User Confirmation
  ```
  ❌ Delete accounts or user data
  ❌ Make purchases or payments
  ❌ Send real messages/emails
  ❌ Change security/permission settings
  ❌ Install software
  ```
  