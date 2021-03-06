project(
	'sway',
	'c',
	version: '1.1',
	license: 'MIT',
	meson_version: '>=0.48.0',
	default_options: [
		'c_std=c11',
		'warning_level=2',
		'werror=true',
	],
)

add_project_arguments(
	[
		'-DWLR_USE_UNSTABLE',

		'-Wno-unused-parameter',
		'-Wno-unused-result',
		'-Wundef',
		'-Wvla',
	],
	language: 'c',
)

cc = meson.get_compiler('c')

is_freebsd = host_machine.system().startswith('freebsd')
datadir = get_option('datadir')
sysconfdir = get_option('sysconfdir')
prefix = get_option('prefix')

if is_freebsd
	add_project_arguments('-D_C11_SOURCE', language: 'c')
endif

jsonc          = dependency('json-c', version: '>=0.13')
pcre           = dependency('libpcre')
wayland_server = dependency('wayland-server')
wayland_client = dependency('wayland-client')
wayland_cursor = dependency('wayland-cursor')
wayland_egl    = dependency('wayland-egl')
wayland_protos = dependency('wayland-protocols', version: '>=1.14')
xkbcommon      = dependency('xkbcommon')
cairo          = dependency('cairo')
pango          = dependency('pango')
pangocairo     = dependency('pangocairo')
gdk_pixbuf     = dependency('gdk-pixbuf-2.0', required: get_option('gdk-pixbuf'))
pixman         = dependency('pixman-1')
libevdev       = dependency('libevdev')
libinput       = dependency('libinput', version: '>=1.6.0')
systemd        = dependency('libsystemd', version: '>=239', required: false)
elogind        = dependency('libelogind', version: '>=239', required: false)
xcb            = dependency('xcb', required: get_option('xwayland'))
math           = cc.find_library('m')
rt             = cc.find_library('rt')
git            = find_program('git', native: true, required: false)

# Try first to find wlroots as a subproject, then as a system dependency
wlroots_version = ['>=0.7.0', '<0.8.0']
wlroots_proj = subproject(
	'wlroots',
	default_options: ['rootston=false', 'examples=false'],
	required: false,
	version: wlroots_version,
)
if wlroots_proj.found()
	wlroots = wlroots_proj.get_variable('wlroots')
	wlroots_conf = wlroots_proj.get_variable('conf_data')
	wlroots_has_xwayland = wlroots_conf.get('WLR_HAS_XWAYLAND') == 1
else
	wlroots = dependency('wlroots', version: wlroots_version)
	wlroots_has_xwayland = cc.get_define('WLR_HAS_XWAYLAND', prefix: '#include <wlr/config.h>', dependencies: wlroots) == '1'
endif

if get_option('xwayland').enabled() and not wlroots_has_xwayland
	error('Cannot enable Xwayland in sway: wlroots has been built without Xwayland support')
endif
have_xwayland = xcb.found() and wlroots_has_xwayland

tray_deps_found = systemd.found() or elogind.found()
if get_option('tray').enabled() and not tray_deps_found
	error('Building with -Dtray=enabled, but libsystemd and libelogind have not been not found')
endif
have_tray = (not get_option('tray').disabled()) and tray_deps_found

conf_data = configuration_data()

conf_data.set10('HAVE_XWAYLAND', have_xwayland)
conf_data.set10('HAVE_GDK_PIXBUF', gdk_pixbuf.found())
conf_data.set10('HAVE_SYSTEMD', systemd.found())
conf_data.set10('HAVE_ELOGIND', elogind.found())
conf_data.set10('HAVE_TRAY', have_tray)

scdoc = dependency('scdoc', version: '>=1.9.2', native: true, required: get_option('man-pages'))
if scdoc.found()
	scdoc_prog = find_program(scdoc.get_pkgconfig_variable('scdoc'), native: true)
	sh = find_program('sh', native: true)
	mandir = get_option('mandir')
	man_files = [
		'sway/sway.1.scd',
		'sway/sway.5.scd',
		'sway/sway-bar.5.scd',
		'sway/sway-input.5.scd',
		'sway/sway-ipc.7.scd',
		'sway/sway-output.5.scd',
		'swaybar/swaybar-protocol.7.scd',
		'swaymsg/swaymsg.1.scd',
		'swaynag/swaynag.1.scd',
		'swaynag/swaynag.5.scd',
	]
	foreach filename : man_files
		topic = filename.split('.')[-3].split('/')[-1]
		section = filename.split('.')[-2]
		output = '@0@.@1@'.format(topic, section)

		custom_target(
			output,
			input: filename,
			output: output,
			command: [
				sh, '-c', '@0@ < @INPUT@ > @1@'.format(scdoc_prog.path(), output)
			],
			install: true,
			install_dir: '@0@/man@1@'.format(mandir, section)
		)
	endforeach
endif

add_project_arguments('-DSYSCONFDIR="/@0@"'.format(join_paths(prefix, sysconfdir)), language : 'c')

version = '"@0@"'.format(meson.project_version())
if git.found()
	git_commit_hash = run_command([git.path(), 'describe', '--always', '--tags'])
	git_branch = run_command([git.path(), 'rev-parse', '--abbrev-ref', 'HEAD'])
	if git_commit_hash.returncode() == 0 and git_branch.returncode() == 0
		version = '"@0@ (" __DATE__ ", branch \'@1@\')"'.format(git_commit_hash.stdout().strip(), git_branch.stdout().strip())
	endif
endif
add_project_arguments('-DSWAY_VERSION=@0@'.format(version), language: 'c')

# Compute the relative path used by compiler invocations.
source_root = meson.current_source_dir().split('/')
build_root = meson.build_root().split('/')
relative_dir_parts = []
i = 0
in_prefix = true
foreach p : build_root
	if i >= source_root.length() or not in_prefix or p != source_root[i]
		in_prefix = false
		relative_dir_parts += '..'
	endif
	i += 1
endforeach
i = 0
in_prefix = true
foreach p : source_root
	if i >= build_root.length() or not in_prefix or build_root[i] != p
		in_prefix = false
		relative_dir_parts += p
	endif
	i += 1
endforeach
relative_dir = join_paths(relative_dir_parts) + '/'

# Strip relative path prefixes from the code if possible, otherwise hide them.
if cc.has_argument('-fmacro-prefix-map=/prefix/to/hide=')
	add_project_arguments(
		'-fmacro-prefix-map=@0@='.format(relative_dir),
		language: 'c',
	)
else
	add_project_arguments(
		'-DSWAY_REL_SRC_DIR="@0@"'.format(relative_dir),
		language: 'c',
	)
endif


sway_inc = include_directories('include')

subdir('include')
subdir('protocols')
subdir('common')
subdir('sway')
subdir('swaymsg')

subdir('client')
subdir('swaybar')
subdir('swaynag')

config = configuration_data()
config.set('datadir', join_paths(prefix, datadir))
config.set('prefix', prefix)
config.set('sysconfdir', join_paths(prefix, sysconfdir))

configure_file(
	configuration: config,
	input: 'config.in',
	output: '@BASENAME@',
	install_dir: sysconfdir + '/sway'
)

if is_freebsd
	configure_file(
		configuration: config,
		input: 'security.d/10-freebsd.in',
		output: '@BASENAME@',
		install_dir: sysconfdir + '/sway/security.d'
	)
else
	configure_file(
		configuration: config,
		input: 'security.d/00-defaults.in',
		output: '@BASENAME@',
		install_dir: sysconfdir + '/sway/security.d'
	)
endif

install_data(
	'sway.desktop',
	install_dir: datadir + '/wayland-sessions'
)

if get_option('default-wallpaper')
	wallpaper_files = files(
		'assets/Sway_Wallpaper_Blue_768x1024.png',
		'assets/Sway_Wallpaper_Blue_768x1024_Portrait.png',
		'assets/Sway_Wallpaper_Blue_1136x640.png',
		'assets/Sway_Wallpaper_Blue_1136x640_Portrait.png',
		'assets/Sway_Wallpaper_Blue_1366x768.png',
		'assets/Sway_Wallpaper_Blue_1920x1080.png',
		'assets/Sway_Wallpaper_Blue_2048x1536.png',
		'assets/Sway_Wallpaper_Blue_2048x1536_Portrait.png',
	)
	wallpaper_install_dir = datadir + '/backgrounds/sway'

	install_data(wallpaper_files, install_dir: wallpaper_install_dir)
endif

if get_option('zsh-completions')
	zsh_files = files(
		'completions/zsh/_sway',
		'completions/zsh/_swaymsg',
	)
	zsh_install_dir = datadir + '/zsh/site-functions'

	install_data(zsh_files, install_dir: zsh_install_dir)
endif

if get_option('bash-completions')
	bash_files = files(
		'completions/bash/sway',
		'completions/bash/swaybar',
		'completions/bash/swaymsg',
	)
	bash_install_dir = datadir + '/bash-completion/completions'

	install_data(bash_files, install_dir: bash_install_dir)
endif

if get_option('fish-completions')
	fish_files = files(
		'completions/fish/sway.fish',
		'completions/fish/swaymsg.fish',
		'completions/fish/swaynag.fish',
	)
	fish_install_dir = datadir + '/fish/completions'

	install_data(fish_files, install_dir: fish_install_dir)
endif

status = [
	'',
	'Features:',
	'xwayland: @0@'.format(have_xwayland),
	'gdk-pixbuf: @0@'.format(gdk_pixbuf.found()),
	'systemd: @0@'.format(systemd.found()),
	'elogind: @0@'.format(elogind.found()),
	'tray: @0@'.format(have_tray),
	'man-pages: @0@'.format(scdoc.found()),
	'',
]
message('\n'.join(status))

if not systemd.found() and not elogind.found()
	warning('The sway binary must be setuid when compiled without (e)logind')
	warning('You must do this manually post-install: chmod a+s /path/to/sway')
endif
