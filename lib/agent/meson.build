agent_sources = [
  'agent.vala',
  'agent-glue.c',
]

force_cpp_linking = have_v8
if force_cpp_linking
  agent_sources += ['dummy.cpp']
endif

extra_vala_args = []
extra_link_args = []
extra_link_depends = []

if host_os_family != 'windows'
  extra_vala_args += '--pkg=posix'
endif
if host_os_family == 'darwin'
  extra_vala_args += '--pkg=frida-gum-darwin-1.0'
endif

if host_os_family == 'windows'
  if host_toolchain != 'microsoft'
    symfile = (host_arch == 'x86') ? 'frida-agent-x86.symbols' : 'frida-agent.symbols'
    extra_link_args += '-Wl,--retain-symbols-file,' + meson.current_source_dir() / symfile
    extra_link_depends += symfile
  endif
elif host_os_family == 'darwin'
  extra_link_args += '-Wl,-exported_symbol,_frida_agent_main'
else
  symscript = (host_os == 'android') ? 'frida-agent-android.version' : 'frida-agent.version'
  extra_link_args += '-Wl,--version-script,' + meson.current_source_dir() / symscript
  extra_link_depends += symscript
endif

if host_os_family in ['linux', 'qnx']
  # For our libc shim to be effective.
  extra_link_args += '-Wl,-Bsymbolic'
endif

raw_agent = shared_library('frida-agent-raw', agent_sources,
  name_prefix: 'lib',
  c_args: frida_component_cflags,
  vala_header: 'frida-agent.h',
  vala_args: [base_vala_args, payload_vala_args, extra_vala_args],
  link_args: extra_link_args,
  link_depends: extra_link_depends,
  vs_module_defs: 'frida-agent.def',
  override_options: ['b_asneeded=true'],
  dependencies: [gio_dep, gee_dep, gum_dep, gumjs_dep, base_dep, pipe_dep, tls_provider_dep, payload_dep],
)

operations = ['--move', 'constructor', 'frida_init_memory', 'first']
if host_os_family != 'darwin'
  operations += ['--move', 'destructor', 'frida_deinit_memory', 'first']
endif
modulated_agent = custom_target('frida-agent-modulated',
  input: raw_agent,
  output: 'libfrida-agent-modulated' + shlib_suffix,
  command: modulate + operations,
)

need_universal_agent = host_os_family == 'darwin' \
    and get_option('assets') == 'installed' \
    and agent_compat != ''

if host_os_family == 'darwin'
  identity = 'FridaAgent'
else
  identity = agent_name
endif
agent = custom_target('frida-agent',
  input: modulated_agent,
  output: need_universal_agent ? f'@agent_name@-@host_abi@' : agent_name,
  command: post_process + ['shared-library', identity],
  build_by_default: true,
  install: get_option('assets') == 'installed' and not need_universal_agent,
  install_dir: asset_dir,
)

agent_dep = declare_dependency(link_with: agent)

if need_universal_agent
  custom_target('frida-agent-universal',
    input: agent,
    output: agent_name,
    command: [
      lipo,
      '@INPUT@',
      agent_compat,
      '-create',
      '-output', meson.current_build_dir() / agent_name,
    ],
    depends: agent_depends,
    install: true,
    install_dir: asset_dir,
  )
endif
