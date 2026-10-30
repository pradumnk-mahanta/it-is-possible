# it-is-possible

`it-is-possible` is a collection of self-contained technical guides, engineering recipes, and hardware/software workarounds. Each project lives in its own dedicated folder, keeping its documentation, configuration notes, and supporting files self-contained.

## Projects

| Project | Folder | Guide | Description |
| :--- | :--- | :--- | :--- |
| Jio ODU with OpenWrt | [`openwrt-jaf-direct`](openwrt-jaf-direct/) | [README](openwrt-jaf-direct/README.md) | Connect a Jio AirFiber ODU directly to OpenWrt via 464XLAT without the stock IDU |

> The folder name serves as the stable project identifier. A human-readable project title and overview are recorded in the table above.

## Repository Layout

```text
it-is-possible/
|-- README.md                         # Project index and repository conventions
|-- LICENSE                           # License file
`-- <project-name>/
    |-- README.md                     # Project guide and documentation
    `-- ...                           # Scripts, configurations, and supporting assets
```

## Adding a New Project

When contributing or adding a new project guide:

1. Create a dedicated directory: `<project-name>/`
2. Include a comprehensive `README.md` covering background, prerequisites, setup instructions, and troubleshooting.
3. Keep scripts, configurations, and supporting assets inside the project folder.
4. Add the new entry to the [Projects](#projects) table in this `README.md`.

## License

This project is licensed under the [GNU General Public License v3.0](LICENSE).
