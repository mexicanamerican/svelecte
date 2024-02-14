<script context="module">
  import defaults from './settings.js';

  import { escapeHtml } from './lib/utils.js';

  const formatterList = {
    default: function(item) { return escapeHtml(item[this.label]); }
  };
  // TODO: rename to addRenderer
  // provide ability to add additional renderers
  export function addFormatter(name, formatFn) {
    if (name instanceof Object) {
      for (let prop in name) {
        formatterList[prop] = name[prop];
      }
    } else {
      formatterList[name] = formatFn
    }
  };

  export const config = defaults;
  export const TAB_SELECT_NAVIGATE = 'select-navigate';
</script>

<script>
  import { iOS, android, fieldInit, highlightSearch } from './lib/utils.js';
  import { createEventDispatcher, onMount, tick } from 'svelte';
  import { positionDropdown, scrollIntoView, virtualListDimensionsResolver } from './lib/dropdown.js';
  import TinyVirtualList from 'svelte-tiny-virtual-list';
  import { flatList } from './lib/list_new.js';
  import { ensureObjectArray, filterList, getFilterProps, initSelection } from './lib/list.js';
  import { flip } from 'svelte/animate';
  import { bindItem } from './actions.js';

  /** @type {string} */
  export let name = 'svelecte';
  /** @type {string} */
  export let inputId = '';
  /** @type {boolean} */
  export let required = false;
  /** @type {boolean} */
  export let disabled = false;
  /** @type {boolean} */
  export let ce_anchor = false;
  /** @type {array} */
  export let options = [];
  
  /** @type {string} */
  export let valueField = defaults.valueField;
  /** @type {string} */
  export let labelField = defaults.labelField;
  /** @type {string} */
  export let groupLabelField = defaults.groupLabelField;
  /** @type {string} */
  export let groupItemsField = defaults.groupItemsField;
  /** @type {string} */
  export let disabledField = defaults.disabledField;
  /** @type {string} */
  export let placeholder = defaults.placeholder;// UI, UX
  export let searchable = defaults.searchable;
  export let clearable = defaults.clearable;
  export let renderer = null;
  export let disableHighlight = false;
  export let highlightFirstItem = defaults.highlightFirstItem;
  export let selectOnTab = defaults.selectOnTab;
  export let resetOnBlur = defaults.resetOnBlur;
  export let resetOnSelect = defaults.resetOnSelect;
  export let closeAfterSelect = defaults.closeAfterSelect;
  export let dndzone = () => ({ noop: true, destroy: () => {}});
  export let validatorAction = null;

  export let allowInputModeToggle = false;
  // export let dropdownItem = Item;
  // export let controlItem = Item;
  // multiple
  export let multiple = defaults.multiple;
  export let max = defaults.max;
  export let collapseSelection = defaults.collapseSelection;
  export let alwaysCollapsed = defaults.alwaysCollapsed;
  // creating
  export let creatable = defaults.creatable;
  export let creatablePrefix = defaults.creatablePrefix;
  export let allowEditing = defaults.allowEditing;
  export let keepCreated = defaults.keepCreated;
  export let delimiter = defaults.delimiter;
  export let createFilter = null;
  export let createTransform = null;
  // remote
  export let fetch = null;
  export let fetchMode = 'auto';
  export let fetchCallback = defaults.fetchCallback;
  export let fetchResetOnBlur = true;
  export let minQuery = defaults.minQuery;
  // performance
  export let lazyDropdown = defaults.lazyDropdown;
  // virtual list
  export let virtualList = defaults.virtualList;
  export let vlHeight = defaults.vlHeight;
  export let vlItemSize = defaults.vlItemSize;
  // sifter related
  export let searchField = null;
  export let sortField = null;
  export let disableSifter = false;
  // styling
  let className = 'svelecte-control';
  export { className as class};
  export let style = null;
  // i18n override
  export let i18n = null;
  // API: public
  export let readSelection = null;
  /** @type {array|string|number|null} */
  export let value = null;
  export let labelAsValue = false;
  export let valueAsObject = defaults.valueAsObject;
  // export let parentValue = undefined;

  /** ************************************ preparation */
  /* possibility to provide initial (selected) values in `fetch` mode **/
  if (fetch && value && valueAsObject && (!options || (options && options.length === 0))) {
    options = Array.isArray(value) ? value : [value];
  }

  multiple = name && !multiple ? name.endsWith('[]') : multiple;

  let isIOS = null;
  let isAndroid = null;

  /** ************************************ END preparation */

  const itemConfig = {
    optionsWithGroups: false,
    isOptionArray: options && options.length && typeof options[0] !== 'object',
    optionProps: [],
    valueField: valueField,
    labelField: labelField,
    // TODO: DEPRECATE `labelAsValue`
    labelAsValue: labelAsValue,
    optLabel: groupLabelField,
    optItems: groupItemsField
  }

  const parentValue = 'UNDEFINED';   // TODO: #178

  let prev_value;
  let prev_options = options;
  let prev_parent_value = parentValue;
  let dropdown_index = highlightFirstItem ? 0 : null;
  let currentValueField = valueField || fieldInit('value', options, itemConfig);
  let currentLabelField = labelField || fieldInit('label', options, itemConfig);
  
  itemConfig.valueField = currentValueField;
  itemConfig.labelField = currentLabelField;
  itemConfig.optionProps = value && valueAsObject && (multiple && Array.isArray(value) ? value.length > 0 : true)
    ? getFilterProps(Array.isArray(value) ? value.slice(0,1).shift() : value)
    : [currentValueField, currentLabelField];

  let selectedOptions = value !== null ? initSelection(options, value, valueAsObject, itemConfig) : [];
  let selectedKeys = selectedOptions.reduce((/** @type {Set} */ set,/** @type {object} */ opt) => { set.add(opt[currentValueField]); return set; }, new Set());
  let alreadyCreated = selectedOptions.filter(opt => opt.$created);

  $: maxReached = max && selectedOptions == max;     // == is intentional, if string is provided

  $: watch_options(options);
  $: options_flat = flatList(options, itemConfig);
  // TODO: add #205 
  $: options_filtered = maxReached
    ? []
    : filterList(options_flat, disableSifter ? null : input_value, multiple ? selectedKeys : false, searchField, sortField, itemConfig);

  const DOM_ID = `sv-select-${name && name !== 'svelecte' ? name : Math.random().toString(36).substring(2)}`;
  const dispatch = createEventDispatcher();


  let is_mounted = false;
  let is_focused = false;
  let is_dropdown_opened = false;

  let ref_select_element = null;

  
  /** 
   * @callback RenderFunction
   * @param {object} item
   * @param {boolean?} [isSelected]
   * @param {string} [inputValue]
   * 
   * @type {RenderFunction}
   */
  $: itemRenderer = typeof renderer === 'function'
    ? renderer
    : (formatterList[renderer] || formatterList.default.bind({ label: currentLabelField}));

  $: watch_value_change(value)
  $: watch_parentValue(parentValue);
  $: watch_selectedOptions(selectedOptions);

  // #region watch functions
  function watch_options(opts) {
    if (is_mounted && prev_options !== opts && opts.length) {
      // make sure, it's an array
      opts = ensureObjectArray(opts);

      const ivalue = fieldInit('value', opts || null, itemConfig);
      const ilabel = fieldInit('label', opts || null, itemConfig);
      if (!valueField && currentValueField !== ivalue) itemConfig.valueField = currentValueField = ivalue;
      if (!labelField && currentLabelField !== ilabel) {
        itemConfig.labelField = currentLabelField = ilabel
        if (renderer === null || renderer === 'default') {
          itemRenderer = formatterList.default.bind({ label: currentLabelField });
        }
      };
      options = opts;
      prev_options = opts;
    }
  }

  /**
   * TODO: extend to allow value out of range
  */
  function watch_value_change(passedVal) {
    if (prev_value === passedVal) return;
    clearSelection();
    if (passedVal) {
      let _selection = Array.isArray(passedVal) ? passedVal : [passedVal];
      const valueProp = itemConfig.labelAsValue ? currentLabelField : currentValueField;
      _selection = _selection.reduce((res, val) => {
        if (creatable && valueAsObject && val.$created) {
          res.push(val);
          return res;
        }
        const opt = options_flat.find(item => valueAsObject
          ? item[currentValueField] == val[currentValueField]
          : item[currentValueField] == val
        );
        opt && res.push(opt);
        return res;
      }, []);
      let success = _selection.every(selectOption) && (multiple
        ? passedVal.length === _selection.length
        : _selection.length > 0
      );
      if (!success) {
        // this is run only when invalid 'value' is provided, like out of option array
        console.warn('[Svelecte]: provided "value" property is invalid', passedVal);
        value = multiple ? [] : null;
        readSelection = value;
        dispatch('invalidValue', passedVal);
        return;
      }
      readSelection = Array.isArray(passedVal) ? _selection : _selection.shift();
    }
    prev_value = passedVal;
  }

  function watch_selectedOptions(newSelection) {
    const selection_formatted = newSelection
      .map(opt => {
        const { '$disabled': unused1,  '$isGroupItem': unused2, ...obj } = opt;
        return obj;
      });
    const unified_selection = multiple
      ? selection_formatted
      : (selection_formatted.length ? selection_formatted[0] : null);
    const valueProp = itemConfig.labelAsValue ? currentLabelField : currentValueField;

    if (!valueAsObject) {
      prev_value = multiple
        ? unified_selection.map(opt => opt[valueProp])
        : selectedOptions.length ? unified_selection[valueProp] : null;
    } else {
      prev_value = unified_selection;
    }
    value = prev_value;
    readSelection = unified_selection;
  }

  function watch_parentValue(newParentValue) {
    if (prev_parent_value !== newParentValue) {
      // TODO: handle value update
      // handleValueUpdate();
      disabled = !parentValue ? true : false;
    }
    prev_parent_value = newParentValue;
  }
  // #endregion

  // #region event-emitters
  /**
   * Dispatch change event on add options/remove selected items
   */
   function emitChangeEvent() {
    tick().then(() => {
      dispatch('change', readSelection);
      if (ref_select_element) {
        ref_select_element.dispatchEvent(new Event('input'));   // required for svelte-use-form
        ref_select_element.dispatchEvent(new Event('change'));  // typically you expect change event to be fired
      }
    });
  }

  /**
   * Dispatch createoption event when user creates a new entry (with 'creatable' feature)
   */
  function emitCreateEvent(createdOpt) {
      dispatch('createoption', createdOpt)
  }
  // #endregion

  /** 
   * Add given option to selection pool
   * Check if not already selected or max item selection reached
   * 
   * @returns bool
   */
   function selectOption(opt) { 
    if (!opt || (multiple && maxReached)) return false;
    if (selectedKeys.has(opt[currentValueField])) return;

    if (typeof opt === 'string') {
      if (!creatable) return;
      opt = createFilter(opt, options);
      if (alreadyCreated.includes(opt)) return;
      !fetch && alreadyCreated.push(opt);
      opt = createTransform(opt, creatablePrefix, currentValueField, currentLabelField);
      opt.$created = true;  // internal setter
      if (keepCreated) options = [...options, opt];
      emitCreateEvent(opt);
    }
    if (multiple) {
      selectedOptions.push(opt);
      selectedOptions = selectedOptions;
      selectedKeys.add(opt[currentValueField]);
    } else {
      selectedOptions = [opt];
      selectedKeys.clear();
      selectedKeys.add(opt[currentValueField]);
      dropdown_index = options.indexOf(opt);
    } 
    options_flat = options_flat;
    return true;
  }

  /**
   * Remove option/all options from selection pool
   */
  function deselectOption(opt, backspacePressed) {
    if (opt.$created && backspacePressed && allowEditing) {
      alreadyCreated.splice(alreadyCreated.findIndex(o => o === opt[labelAsValue ? currentLabelField : currentValueField]), 1);
      alreadyCreated = alreadyCreated;
      if (keepCreated) {
        options.splice(options.findIndex(o => o === opt), 1);
        options = options;
      }
      input_value = opt[currentLabelField].replace(creatablePrefix, '');
    }
    const id = opt[currentValueField];
    selectedKeys.delete(id);
    selectedOptions.splice(selectedOptions.findIndex(o => o[currentValueField] == id), 1);
    selectedOptions = selectedOptions;
    // TODO: check if re-filter items wouldn't be better
    options_flat = options_flat;
  }

  function clearSelection() {
    selectedKeys.clear();
    selectedOptions = [];
    maxReached = false;       // reset forcefully, related to #145
    // TODO: check if re-filter items wouldn't be better
    options_flat = options_flat;  
  }

  /**
   * Handle user action on select
   */
   function onSelect(event, opt) {
    opt = opt || event.detail;
    if (disabled || opt[disabledField] || opt.$isGroupHeader) return;

    selectOption(opt);
    if ((multiple && resetOnSelect) || !multiple) input_value = '';
    if (!multiple || closeAfterSelect) {
      is_dropdown_opened = false;
    } else {
      // TODO: investigate
      // tick().then(() => {
      //   dropdown_index = maxReached
      //     ? null
      //     : listIndex.next(dropdown_index - 1, true);
      // })
    }
    emitChangeEvent();
  }

  function onDeselect(event = {}, opt = null, backspacePressed) {
    if (disabled) return;
    opt = opt || event.detail;
    if (opt) {
      deselectOption(opt, backspacePressed);
    } else {  // apply for 'x' when clearable:true || ctrl+backspace || ctrl+delete
      clearSelection();
    }
    // TODO: investigate
    // tick().then(focusControl);
    emitChangeEvent();
  }

  /**
   * @typedef {object} DirectionSettings
   * @property {boolean} [asc]
   * @property {boolean} [desc]
   * 
   * @param {number} pos
   * @param {DirectionSettings} direction
   */
  function setDropdownIndex(pos, direction = {}, limit = 0) {
    if (limit > 3) return;
    if (pos < 0) pos = direction.desc
      ? options_filtered.length - 1
      : 0;
    if (dropdown_index === null || pos >= options_filtered.length) {
      pos = 0;
    }
    // if pos represents group header, move to next one
    console.log(pos);
    if (options_filtered[pos].$isGroupHeader) {
      setDropdownIndex(direction.asc
        ? pos + 1
        : pos - 1, direction, ++limit
      );
      return;
    }
    dropdown_index = pos;
  }

  /**
   * @param {KeyboardEvent} event
   * 
   * //NOTE: previously Svelecte.svelte/onKeyDown
   */
  function processKeyDown(event) {
    // DEPRECATED check this?
    // event = event.detail; // from dispatched event

    if (creatable && delimiter.indexOf(event.key) > -1) {
      input_value.length > 0 && onSelect(null, input_value); // prevent creating item with delimiter itself
      event.preventDefault();
      return;
    }
    const Tab = selectOnTab && is_dropdown_opened && !event.shiftKey ? 'Tab' : 'No-tab';
    let ctrlKey = isIOS ? event.metaKey : event.ctrlKey;
    let isPageEvent = ['PageUp', 'PageDown'].includes(event.key);
    let backspacePressed = false;
    switch (event.key) {
      case 'End':
        if (input_value.length !== 0) return;
        setDropdownIndex(options_filtered.length);
      // TODO: impl
      // case 'PageDown':
      //   if (isPageEvent) {
      //     const [wrap, item] = refDropdown.getDimensions();
      //     dropdown_index = Math.ceil((item * dropdown_index + wrap) / item);
      //   }
      case 'ArrowUp':
        event.preventDefault();
        if (!is_dropdown_opened) {
          is_dropdown_opened = true;
          // TODO:
          // if (dropdown_index === null) {            
          //   dropdown_index = listIndex.first
          // }
          return;
        }
        setDropdownIndex(dropdown_index - 1, { desc: true });
        // tick().then(refDropdown.scrollIntoView);
        // ignoreHover = true;
        break;
      case 'Home':
        if (input_value.length !== 0
          || (input_value.length === 0 && options_filtered.length === 0)  // ref #26
        ) return;
        setDropdownIndex(0);
      // TODO: impl
      // case 'PageUp':
      //   if (isPageEvent) {
      //     const [wrap, item] = refDropdown.getDimensions();
      //     dropdown_index = Math.floor((item * dropdown_index - wrap) / item);
      //   }
      case 'ArrowDown':
        event.preventDefault();
        if (!is_dropdown_opened) {
          is_dropdown_opened = true;
          // TODO:
          // if (dropdown_index === null) {
          //   dropdown_index = listIndex.first
          // }
          return;
        }
        setDropdownIndex(dropdown_index + 1);
        // TODO:
        // tick().then(refDropdown.scrollIntoView);
        // ignoreHover = true;
        break;
      case 'Escape':
        if (is_dropdown_opened) { // prevent ESC bubble in this case (interfering with modal closing etc. (bootstrap))
          event.preventDefault();
          event.stopPropagation();
        }
        if (!input_value) {
          is_dropdown_opened = false;
        }
        // TODO:
        // cancelXhr();
        input_value = '';
        break;
      case Tab:
      case 'Enter':
        if (!is_dropdown_opened) {
          event.key !== Tab && dispatch('enterKey', event); // ref #125
          return;
        }
        let activeDropdownItem = !ctrlKey ? options_filtered[dropdown_index] : null;
        if (creatable && input_value) {
          activeDropdownItem = !activeDropdownItem || ctrlKey
            ? input_value
            : activeDropdownItem
          ctrlKey = false;
        }
        !ctrlKey && activeDropdownItem && onSelect(null, activeDropdownItem);
        if (options_filtered.length <= dropdown_index) {
          setDropdownIndex(options_filtered.length - 1);
        }
        if (!activeDropdownItem && selectedOptions.length) {
          is_dropdown_opened = false;
          event.key !== Tab && dispatch('enterKey', event); // ref #125
          return;
        }
        (event.key !== Tab || (event.key === Tab && selectOnTab !== TAB_SELECT_NAVIGATE)) && event.preventDefault(); // prevent form submit
        break;
      case ' ':
        if (!fetch && !is_dropdown_opened) {
          is_dropdown_opened = true;
          event.preventDefault();
        }
        // FUTURE: remove to allow search while selected for single select
        if (!multiple && selectedOptions.length) event.preventDefault();
        break;
      case 'Backspace':
        backspacePressed = true;
      case 'Delete':
        if (input_value === '' && selectedOptions.length) {
          ctrlKey ? onDeselect({ /** no detail prop */}) : onDeselect(null, selectedOptions[selectedOptions.length - 1], backspacePressed);
          event.preventDefault();
        }
      default:
        // TODO: keep this old impl, remove line below it
        // if (!ctrlKey && !['Tab', 'Shift'].includes(event.key) && !is_dropdown_opened && !isFetchingData) {
        if (!ctrlKey && !['Tab', 'Shift'].includes(event.key) && !is_dropdown_opened) {
          is_dropdown_opened = true;
        }
        // FUTURE: remove to allow search while selected for single select
        if (!multiple && selectedOptions.length && event.key !== 'Tab') {
          event.preventDefault();
          event.stopPropagation();
        }
    }
  }

  /**
   * Single click handler. Unified for dropdown items, for selected items and 'x' clear button
   * 
   * @param {MouseEvent & { currentTarget: EventTarget & HTMLDivElement} & { target: HTMLElement }} event
   */
  function onClick(event) {
    event.preventDefault();

    /** @type {HTMLElement & import('./actions.js').ExtButton} */
    const target = event.target.closest('[data-action]');
    /** @type {HTMLElement} */
    const dropdown_item = event.target.closest('[data-pos]');

    // handle click on selection row (general focus & toggle dropdown event)
    if (!target && !dropdown_item) {
      return focusControl(event.target);
    }
    const action = target?.dataset.action || 'select';
    
    // dropdown items has no data-action set
    switch(action) {
      case 'deselect':
        onDeselect({}, target.bound_item)
        break;
      case 'select': 
        const opt_position = parseInt(dropdown_item.dataset.pos);
        onSelect(null, options_filtered[opt_position]);
        break;
      case 'toggle':
        is_dropdown_opened = !is_dropdown_opened;
        break;
    }
  }

  /**
   * @param {HTMLElement} target
   */
  function focusControl(target) {
    if (disabled) return;
    if (!is_focused) {
      ref_input.focus();
    } else {
      if ((isAndroid || isIOS) && input_mode !== 'text') {
        input_mode = 'text';
        return;
      }
      if (target.tagName === 'INPUT' && input_value) {
        return;
      } else if (input_mode === 'text') {
        input_mode = 'none';
        is_dropdown_opened = true;
        return;
      }
      is_dropdown_opened = !is_dropdown_opened;
    }
  }

  //#region dropdown
  /** ************************************ dropdown-related */

  let render_dropdown = !lazyDropdown;
  let dropdown_scroller = null;
  let ref_container;
  let ref_container_scroll;
  let ref_virtuallist = null;
  let virtuallist_automode = virtualList && vlHeight === null && vlItemSize === null;
  let vl_height = vlHeight;
  let vl_itemSize = vlItemSize;
  let hasEmptyList = false;

  $: vl_listHeight = Math.min(vl_height, Array.isArray(vl_itemSize) 
    ? vl_itemSize.reduce((res, num) => {
      res+= num;
      return res;
    }, 0)
    : options_filtered.length * vl_itemSize
  );
  // TODO: refactor
  $: virtuallist_automode && watch_options_virtualList(options);

  function watch_options_virtualList(options) {
    if (!is_mounted || !render_dropdown) return;
    // required when changing item list 'on-the-fly' for VL
    // if (hasEmptyList) dropdown_index = null;
    tick()
      .then(() => {
        const dimensions = virtualListDimensionsResolver(ref_virtuallist, ref_container_scroll, options_filtered);
        vl_itemSize = dimensions.size;
        vl_height = dimensions.height;
      })
      .then(() => positionDropdown(is_dropdown_opened, ref_container_scroll, render_dropdown));
  }
  
  $: watch_is_dropdown_opened(is_dropdown_opened);

  function watch_is_dropdown_opened(val) {
    if (!is_mounted) return;

    if (!render_dropdown && val) render_dropdown = true;
    tick()
      .then(() => {
        virtuallist_automode && watch_options_virtualList([]);
      })
      .then(() => {
        positionDropdown(val, ref_container_scroll, true);
        val && scrollIntoView({ container: ref_container, scrollContainer: ref_container_scroll, virtualList, center: true }, dropdown_index);
      });

    if (!dropdown_scroller) dropdown_scroller = () => positionDropdown(val, ref_container_scroll, true);
    // bind/unbind scroll listener
    document[val ? 'addEventListener' : 'removeEventListener']('scroll', dropdown_scroller, { passive: true });
  }

  //#endregion

  //#region input
  /** ************************************ input-related */

  /** @type {string} */
  let input_value = '';
  let disable_key_event_bubble = false;
  /** @type {HTMLInputElement} */
  let ref_input = null;
  
  /** @type {"none"|"text"} */
  $: input_mode = allowInputModeToggle
    // TODO: implement this
    ? 'text' // in the meantime it has default behavior
    : 'text'
  /** @type {string} */
  $: placeholder_active = selectedOptions.length ? '' : placeholder;
  /** @type {'enter'} */
  $: enter_hint = selectedOptions.length > 0 && multiple === false ? null : 'enter';

  /**
   * @param {KeyboardEvent} e
   */
   function onKeyDown(e) {
    if (android() && !enter_hint && e.key === 'Enter') return true;

    disable_key_event_bubble = ['Enter', 'Escape'].includes(e.key) && is_dropdown_opened;
    processKeyDown(e);
  }

  /**
   * @param {KeyboardEvent} e
   */
  function onKeyUp(e) {
    if (disable_key_event_bubble) {
      e.stopImmediatePropagation();
      e.preventDefault();
    }
    disable_key_event_bubble = false;
  }

  /**
   * Required for mobile single select to work properly
   */
  function onInput() {
    if (selectedOptions.length === 1 && !multiple) {
      // input_value = '';
    }
  }

  // TODO: missing functionality related to AlwayCollapsed
  function onFocus() {
    is_focused = true;
    is_dropdown_opened = true;
  }

  // TODO: missing functionality related to AlwayCollapsed
  function onBlur() {
    is_focused = false;
    is_dropdown_opened = false;
    if (resetOnBlur) input_value = '';
  }

  // #endregion

  onMount(() => {
    is_mounted = true;
    isAndroid = android();
    isIOS = iOS();
  });

  // NOTE: remove when refactored
  let doCollapse = false;
  let isFetchingData = false;

  console.log('I', selectedOptions, value);
</script>

<div class="svelecte">
<!-- svelte-ignore a11y-no-static-element-interactions -->
<!-- svelte-ignore a11y-click-events-have-key-events -->
<div class="sv-control" on:mousedown={onClick}>
    <slot name="icon"></slot>
    <!-- selection & input -->
    <div class="sv_selection-wrap" class:is-single={multiple === false}>
      {#if selectedOptions.length }
      <!-- TODO: re-implement comments -->
      <!-- {#if multiple && collapseSelection && doCollapse} -->
        <!-- {@html collapseSelection(selectedOptions.length, selectedOptions) } -->
      <!-- {:else} -->
          {#each selectedOptions as opt (opt[currentValueField])}
          <div class="sv-flex-ellipsis" class:is-multi={multiple} animate:flip={{duration: 100 }}>
            <div class="sv-item-content">{@html itemRenderer(opt, true)}</div>
            {#if multiple}
            <button class="sv-item-btn" tabindex="-1" type="button"
              data-action="deselect"
              use:bindItem={opt}
              on:mousedown|preventDefault
            >
              <svg height="16" width="16" viewBox="0 0 20 20" aria-hidden="true" focusable="false">
                <path d="M14.348 14.849c-0.469 0.469-1.229 0.469-1.697 0l-2.651-3.030-2.651 3.029c-0.469 0.469-1.229 0.469-1.697 0-0.469-0.469-0.469-1.229 0-1.697l2.758-3.15-2.759-3.152c-0.469-0.469-0.469-1.228 0-1.697s1.228-0.469 1.697 0l2.652 3.031 2.651-3.031c0.469-0.469 1.228-0.469 1.697 0s0.469 1.229 0 1.697l-2.758 3.152 2.758 3.15c0.469 0.469 0.469 1.229 0 1.698z"></path>
              </svg>
            </button>
            {/if}
          </div>
          {/each}
        <!-- {/if} -->
      {/if}

      <!-- #regions INPUT -->
      <span class="sv_input-sizer" data-value={input_value || placeholder_active}>
        <input type="text" class="sv_input-text" size="1"
          placeholder={input_value ? '' : placeholder_active}
          inputmode={input_mode}
          enterkeyhint={enter_hint}
          bind:this={ref_input}
          bind:value={input_value}
          on:focus={onFocus}
          on:keydown={onKeyDown}
          on:keyup={onKeyUp}
          on:input={onInput}
          on:blur={onBlur}
          >
      </span>
      <!-- #endregion -->
    </div>

    <!-- buttons, indicators -->
    <div class="sv-buttons" class:is-loading={isFetchingData}>
      {#if clearable && !disabled}
      <button type="button" class="sv-btn is-indicator" class:sv-has-selection={selectedOptions.length}
        data-action="deselect"  tabindex="-1"
        on:mousedown|preventDefault
      >
        <slot name="clear-icon" {selectedOptions} inputValue={input_value}>
          {#if selectedOptions.length}
          <svg class="indicator-icon" viewBox="0 0 20 20" aria-hidden="true" focusable="false"><path d="M14.348 14.849c-0.469 0.469-1.229 0.469-1.697 0l-2.651-3.030-2.651 3.029c-0.469 0.469-1.229 0.469-1.697 0-0.469-0.469-0.469-1.229 0-1.697l2.758-3.15-2.759-3.152c-0.469-0.469-0.469-1.228 0-1.697s1.228-0.469 1.697 0l2.652 3.031 2.651-3.031c0.469-0.469 1.228-0.469 1.697 0s0.469 1.229 0 1.697l-2.758 3.152 2.758 3.15c0.469 0.469 0.469 1.229 0 1.698z"></path></svg>
          {/if}
        </slot>
      </button>
      {/if}
      {#if clearable}<span class="sv-btn-separator"></span>{/if}
      <button type="button" class="sv-btn is-indicator" class:sv-dropdown-opened={is_dropdown_opened}
        data-action="toggle" tabindex="-1"
        on:mousedown|preventDefault
      >
        <!-- TODO: rename this slot -->
        <slot name="indicator-icon" hasDropdownOpened={is_dropdown_opened}>
          <svg class="indicator-icon" viewBox="0 0 20 20" aria-hidden="true" focusable="false">
            <path d="M4.516 7.548c0.436-0.446 1.043-0.481 1.576 0l3.908 3.747 3.908-3.747c0.533-0.481 1.141-0.446 1.574 0 0.436 0.445 0.408 1.197 0 1.615-0.406 0.418-4.695 4.502-4.695 4.502-0.217 0.223-0.502 0.335-0.787 0.335s-0.57-0.112-0.789-0.335c0 0-4.287-4.084-4.695-4.502s-0.436-1.17 0-1.615z"></path>
          </svg>
        </slot>
      </button>
    </div>
    <slot name="control-end"></slot>
</div>

<!-- #region DROPDOWN -->
  <!-- svelte-ignore a11y-no-static-element-interactions -->
  <!-- svelte-ignore a11y-click-events-have-key-events -->
  <div class="sv_dropdown" class:is-open={is_dropdown_opened}
    on:mousedown={onClick}
  >
    
  {#if is_mounted && render_dropdown}
    <!-- 
      TODO: selection rendering in dropdown 
      NOTE: BC 
      {#if isInitialized && collapseSelection && alwaysCollapsed && selectedOptions.length}
      <div class="sv-content has-multiSelection alwaysCollapsed-selection">
        {#each selectedOptions as opt, i (i)}
          <svelte:component this={controlItem} formatter={itemRenderer} item={opt} isSelected={true} on:deselect isMultiple={multiple} {inputValue}/>
        {/each}
      </div>
      {/if}
    -->
    <div bind:this={ref_container_scroll} class="sv-dropdown-scroll" class:is-empty={!options_filtered.length} class:is-virtual={virtualList} tabindex="-1">
      <div bind:this={ref_container} class="sv-dropdown-content" class:max-reached={maxReached} >
      {#if options_filtered.length}
      {#if virtualList}
          <TinyVirtualList bind:this={ref_virtuallist}
            width="100%"
            height={vl_listHeight}
            itemCount={options_filtered.length}
            itemSize={vl_itemSize}
            scrollToAlignment="auto"
            scrollToIndex={dropdown_index}
          >
            <div slot="item" let:index let:style {style}>
              {@const opt = options_filtered[index]}
              {#if opt.$isGroupHeader}
                <div class="sv-optgroup-header"><b>{opt.label}</b></div>
              {:else}
                <div data-pos={index} 
                  class="sv-dropdown-item sv-flex-ellipsis"
                  class:sv-dd-item-active={dropdown_index === index}
                >
                  <div class="sv-item-content">
                    {@html highlightSearch(opt, false, input_value, itemRenderer) }
                  </div>
                </div>
              {/if}
            </div>
          </TinyVirtualList>
        {:else}
          {#each options_filtered as opt, i}
            {#if opt.$isGroupHeader}
              <div class="sv-optgroup-header"><b>{opt.label}</b></div>
            {:else}
              <div data-pos={i} 
                class="sv-dropdown-item sv-flex-ellipsis"
                class:sv-dd-item-active={dropdown_index === i}
              >
                <div class="sv-item-content">
                  {@html highlightSearch(opt, false, input_value, itemRenderer) }
                </div>
              </div>
            {/if}
          {/each}
        {/if}
      {/if}
      {#if options_filtered.length === 0 || maxReached}
        <!-- TODO: listMessage -->
        <div class="empty-list-row">Empty list</div>
      {/if}
    </div>
  </div> <!-- scroll container end -->
  {#if creatable && input_value && !maxReached}
    <div class="creatable-row-wrap">
      <!-- svelte-ignore a11y-click-events-have-key-events -->
      <!-- TODO: re-impl create row -->
      <!-- <div class="creatable-row" on:click={dispatch('select', inputValue)} on:mouseenter={dispatch('hover', listIndex.last)}
        class:active={currentListLength == dropdownIndex}
        class:is-disabled={alreadyCreated.includes(inputValue)}
      >
      {@html createLabel(inputValue)}
      {#if currentListLength != dropdownIndex}
        <span class="shortcut"><kbd>{metaKey}</kbd>+<kbd>Enter</kbd></span>
      {/if}
      </div> -->
    </div>
  {/if}
  {/if}
<!-- #endregion -->
</div>

</div> <!-- /svelecte -->

<style>
  /** NOTE: WIP */
  .svelecte {
    --sv-border: var(--sv-border-width, 1px) var(--sv-border-style, solid) var(--sv-border-color, #ccc);
    position: relative;
    flex: 1 1 auto;
    color: var(--sv-color, inherit);
    --sv-general-padding: 4px;  /* theme */
    --sv-input-wrap-padding: var(--sv-general-padding);
    --sv-item-padding: var(--sv-general-padding);
    --sv-dropdown-offset: 1px;
  }

  .sv-control {
    display: flex;
    align-items: center;
    border: 1px solid #ccc;
    border-radius: 4px;
  }
  
  .sv_selection-wrap {
    display: flex;
    flex-wrap: wrap;
    align-items: center;
    min-width: 0;
    flex: 1;
    padding: var(--sv-general-padding, 4px);
    gap: 4px;
    min-height: 24px;
    &.is-single {
      flex-wrap: nowrap;
    }

    & .sv-flex-ellipsis {
      padding: 3px 3px 3px 6px;
    }
  }
  .sv-flex-ellipsis {
    display: flex;
    min-width: 0;
    &.is-multi {
      background-color: var(--sv-item-selected-bg, #efefef);
    }
  }
  .sv-item-content {
    overflow: hidden;
    white-space: nowrap;
    text-overflow: ellipsis;
  }

  .sv-item-btn {
    position: relative;
    display: inline-flex;
    align-items: center;
    align-self: stretch;
    padding: 0 4px;
    box-sizing: border-box;
    border-radius: 2px;
    border-width: 0;
    margin: 0;
    cursor: pointer;
    background-color: var(--sv-item-btn-bg, var(--sv-item-selected-bg, #efefef));
  }
  .sv-item-btn:hover {
    background-color: var(--sv-item-btn-bg-hover, #ddd);
  }
  .sv-item-btn > svg {
    fill: var(--sv-item-btn-icon, var(--sv-icon-color));
  }

  /** ************************************ buttons */

  .sv-buttons {
    display: flex;
    align-self: stretch;
    position: relative;
  }
  .sv-btn {
    color: var(--sv-icon-color, #bbb);
    display: flex;
    transition: color 150ms ease 0s;
    box-sizing: border-box;
    background-color: var(--sv-icon-bg-color, transparent);
    border-width: var(--sv-icon-border-width, 0);
    padding: 0;
    margin: var(--sv-general-padding, 4px);
  }
  .sv-btn:hover {
    color: var(--sv-icon-color-hover, #777);
  }
  .sv-btn.is-indicator {
    align-items: center;
    fill: currentcolor;
    line-height: 1;
    stroke: currentcolor;
    stroke-width: 0px;
  }
  .sv-btn-separator {
    align-self: stretch;
    background-color: var(--sv-border-color, #ccc);
    margin-bottom: var(--sv-general-padding, 4px);
    margin-top: var(--sv-general-padding, 4px);
    width: 1px;
    box-sizing: border-box;
  }
  .indicator-icon {
    width: var(--sv-icon-size, 20px);
    height: var(--sv-icon-size, 20px);
  }
  .is-loading:after {
    animation: spinAround 0.5s infinite linear;
    border: var(--sv-loader-border, 2px solid #ccc);
    border-radius: 290486px;
    border-right-color: transparent;
    border-top-color: transparent;
    content: "";
    display: block;
    height: var(--sv-icon-size, 20px);
    width: var(--sv-icon-size, 20px);
    right: var(--sv-general-padding, 4px);
    top: calc(50% - (var(--sv-icon-size, 20px) / 2));
    position: absolute !important;
    box-sizing: border-box;
  }
  @keyframes spinAround {
    from {
      transform: rotate(0deg)
    }
    to {
      transform: rotate(359deg)
    }
  }

  /** ************************************ dropdown */

  .sv_dropdown {
    margin: var(--sv-dropdown-offset, 0) 0;
    box-sizing: border-box;
    position: absolute;
    min-width: 100%;
    width: var(--sv-dropdown-width);
    display: none;
    background-color: var(--sv-bg, #fff);
    overflow-y: auto;
    overflow-x: hidden;
    border: 1px solid rgba(0,0,0,0.15);
    border-radius: var(--sv-border-radius, 4px);
    box-shadow: var(--sv-dropdown-shadow, 0 6px 12px #0000002d);
    z-index: 2;
  }
  .sv_dropdown.is-open {
    display: block;
  }
  .sv-dropdown-scroll {
    /* min-height: 40px; */
    padding: 4px;
    box-sizing: border-box;
    max-height: var(--sv-dropdown-height, 316px);
    overflow-y: auto;
    overflow-x: hidden;
  }
  /* .sv-dropdown-content {
    padding: var(--sv-general-padding);
  } */
  .sv-dd-item-active,
  .sv-dropdown-item:hover,
  .sv-dropdown-item:active {
    background-color: #F2F5F8;
  }

  /** #region input */
  .sv_input-sizer {
    display: inline-grid;
    vertical-align: top;
    align-items: center;
    position: relative;

    &::after {
      content: attr(data-value) ' ';
      visibility: hidden;
      white-space: pre-wrap;
    }

    &:not(:focus-within) {
      position: absolute;
    }
  }
  .sv_input-sizer:after,
  .sv_input-text {
    width: auto;
    min-width: 1em;
    grid-area: 1 / 2;
    font: inherit;
    padding: 0 0.25em;
    margin: 0;
    resize: none;
    background: none;
    appearance: none;
    border: none;
  }
  .sv_input-text {
    outline: none;
    &:placeholder {
      color: var(--sv-placeholder-color, #ccccd6);
    }
  }
  /* #endregion */
  /** NOTE: DEV ONLY */
  .sv_input-text {
    border: 1px dotted red;
  }
</style>
